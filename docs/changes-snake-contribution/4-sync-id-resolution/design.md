# Design: Sync ID Resolution & LWW

## Technical Approach

Refactorizar `Import()` para reemplazar inserciones ciegas por un bucle condicional con resolución LWW por `sync_id`. Normalizar `newSyncID()` a formato único. Expandir `ImportResult` con contadores granulares. Las comprobaciones ocurren dentro de la misma `tx` existente.

## Architecture Decisions

### Decision: Lógica LWW manual vs constraint UNIQUE + ON CONFLICT

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| LWW manual con `SELECT` + `INSERT/UPDATE` | Más código, pero no requiere migración de schema | ✅ Elegida |
| Añadir `UNIQUE(sync_id)` + `ON CONFLICT` | Limpio en SQL, pero requiere migrar datos legacy (posibles duplicados existentes) | Descartada |

**Rationale**: No hay garantía de que `sync_id` sea único en DBs existentes (el backfill genera 32 hex, el código Go genera 16 hex — podrían existir colisiones). Una migración `UNIQUE` podría fallar en producción. La lógica manual es segura.

### Decision: Normalizar formato de sync_id

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| Estandarizar a `{prefix}-{16hex}` (formato Go) | Consistencia futura; el backfill ya pasó, no hay urgencia de migrar existentes | ✅ Elegida |
| UUIDv7 | Introduciría un tercer formato incompatible | Descartada |
| Estandarizar a `{prefix}-{32hex}` (formato backfill) | Mayor entropía pero inconsistente con el código Go existente | Descartada |

**Rationale**: Normalizar solo la generación futura. Los `sync_id` existentes (ambos formatos) se aceptan tal cual. `normalizeExistingSyncID()` (línea 5310) ya maneja la coexistencia.

### Decision: Preservar deleted_at en LWW

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| `deleted_at` participa en UPDATE como campo regular | Un import con obs soft-deleted más reciente puede soft-deletear una obs local activa | ✅ Elegida |
| Ignorar `deleted_at` del payload | Importar un dump no propagaría borrados legítimos | Descartada |

**Rationale**: LWW puro — el campo `deleted_at` es parte del estado de la observación. Si el payload es más nuevo, TODO su estado (incluyendo borrado) prevalece.

### Decision: Prompts → SKIP (no LWW)

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| Si existe sync_id → SKIP | Inmutables, no hay `updated_at` para comparar | ✅ Elegida |
| Intentar LWW con `created_at` | Abuso semántico — `created_at` no indica "última versión" | Descartada |

## Data Flow

```
Import(data *ExportData)
    │
    └── s.withTx(func(tx) {
            // Sessions: INSERT OR IGNORE (sin cambios)
            for obs in data.Observations:
              SELECT id, updated_at, deleted_at FROM observations WHERE sync_id = ? LIMIT 1
              ├── ErrNoRows → INSERT (con deleted_at del payload)
              ├── Payload.UpdatedAt >  Local → UPDATE (todos los campos incl. deleted_at)
              └── Payload.UpdatedAt <= Local → SKIP
            for prompt in data.Prompts:
              SELECT id FROM user_prompts WHERE sync_id = ? LIMIT 1
              ├── ErrNoRows → INSERT
              └── Exists → SKIP
        })
    │
    └── ImportResult{Inserted, Updated, Skipped por entidad}
```

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `internal/store/store.go:2836-2862` | Modify | Reemplazar INSERT ciego por bucle LWW para observations. |
| `internal/store/store.go:2864-2875` | Modify | Reemplazar INSERT ciego por bucle SKIP para prompts. |
| `internal/store/store.go:2884-2888` | Modify | Expandir `ImportResult` con campos granulares. |
| `internal/store/store.go:5302-5307` | Modify | Normalizar `newSyncID()` a formato consistente (documentar decisión). |

## Interfaces / Contracts

```go
// ImportResult expandido
type ImportResult struct {
    SessionsImported     int `json:"sessions_imported"`
    ObservationsInserted int `json:"observations_inserted"`
    ObservationsUpdated  int `json:"observations_updated"`
    ObservationsSkipped  int `json:"observations_skipped"`
    PromptsInserted      int `json:"prompts_inserted"`
    PromptsSkipped       int `json:"prompts_skipped"`
}
```

La query de resolución para observations:
```sql
SELECT id, updated_at, deleted_at FROM observations WHERE sync_id = ? LIMIT 1
```

El UPDATE LWW replica TODOS los campos del payload:
```sql
UPDATE observations
SET session_id=?, type=?, title=?, content=?, tool_name=?, project=?,
    scope=?, topic_key=?, normalized_hash=?, revision_count=?,
    duplicate_count=?, last_seen_at=?, created_at=?, updated_at=?, deleted_at=?
WHERE id = ?
```

## Testing Strategy

| Layer | What to Test | Approach |
|-------|-------------|----------|
| Unit | Idempotencia de reimport | Import dump → reimport → assert `ObservationsSkipped = N`, `Inserted = 0`. |
| Unit | LWW update (payload más nuevo) | Import obs → avanzar `updated_at` en dump → reimport → assert `ObservationsUpdated = 1`. |
| Unit | LWW skip (local más nuevo) | Import obs → avanzar local `updated_at` → reimport dump original → assert `ObservationsSkipped = 1`. |
| Unit | Prompts inmutables | Import prompt → reimport con contenido diferente → assert `PromptsSkipped = 1`, contenido intacto. |
| Unit | deleted_at preservado en INSERT | Import obs con `deleted_at NOT NULL` → assert DB tiene `deleted_at` preservado. |
| Unit | deleted_at propaga en LWW | Local activa + payload soft-deleted más nuevo → assert local queda `deleted_at NOT NULL`. |
| Unit | Formato sync_id normalizado | Llamar `newSyncID("obs")` → assert formato `obs-{16hex}` consistente. |

## Migration / Rollout

No schema migration required. Los `sync_id` existentes en ambos formatos (16hex y 32hex) se aceptan tal cual — `normalizeExistingSyncID()` ya maneja la coexistencia. Solo se normaliza la generación futura.

## Open Questions

Ninguna.
