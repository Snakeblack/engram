# Design: Consolidación Atómica

## Technical Approach

Nueva función transaccional `Store.Consolidate()` que envuelve soft-deletes + insert en una sola `withTx()`. Nuevo tool MCP `mem_consolidate` registrado en `ProfileAgent` con handler que sigue el patrón exacto de `handleSave`. Cada operación dentro de la tx encola su sync mutation correspondiente.

## Architecture Decisions

### Decision: Función store dedicada vs reutilizar DeleteObservation + AddObservation

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| `Consolidate()` dedicada con `withTx()` | Código nuevo, pero atomicidad garantizada en 1 tx | ✅ Elegida |
| Llamar `DeleteObservation` + `AddObservation` en secuencia | Reutiliza código pero son 2+ tx separadas — no atómico | Descartada |

**Rationale**: La atomicidad es el requisito central. Si el INSERT falla, los deletes se revierten. `withTx` ya garantiza commit/rollback.

### Decision: Soft delete vs hard delete

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| Soft delete (`deleted_at`) | Preserva auditoría + sync propagation | ✅ Elegida |
| Hard delete (`DELETE FROM`) | Irrecuperable, complica sync | Descartada |

**Rationale**: El pipeline de sync espera `SyncOpDelete` con payload que incluye `DeletedAt`. Soft delete mantiene el contrato. Mismo patrón de `DeleteObservation` líneas 2318-2331.

### Decision: Ignorar IDs inexistentes vs retornar error

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| Ignorar silenciosamente | Más robusto; el agente puede pasar IDs obsoletos | ✅ Elegida |
| Error si algún ID no existe | Bloquea la operación por IDs viejos ya purgados | Descartada |

**Rationale**: `applyObservationDeleteTx` ya ignora silenciosamente. Consistencia con patrones existentes.

## Data Flow

```
MCP: mem_consolidate(ids_to_delete, new_title, new_content, ...)
    │
    └── handleConsolidate() → parsea args, resuelve session_id
            │
            └── store.Consolidate(ids, AddObservationParams)
                    │
                    └── s.withTx(func(tx) {
                            for id in ids:
                              getObservationTx(id) → soft-delete → enqueueSyncMutationTx(Delete)
                            INSERT new obs → newSyncID("obs") → enqueueSyncMutationTx(Upsert)
                        })
```

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `internal/store/store.go` | Create func | `Consolidate(idsToDelete []int64, p AddObservationParams) (int64, error)` |
| `internal/mcp/mcp.go` | Modify | Registrar `mem_consolidate` en `ProfileAgent` (línea 70). Añadir handler `handleConsolidate`. |

## Interfaces / Contracts

```go
// store.go — nueva función
func (s *Store) Consolidate(idsToDelete []int64, p AddObservationParams) (int64, error)
// Retorna: ID de la nueva observación, o error si la tx falla.

// mcp.go — parámetros MCP del tool
// ids_to_delete: []int64 (requerido)
// new_title:     string  (requerido)
// new_content:   string  (requerido)
// type:          string  (opcional, default "manual")
// project:       string  (opcional, auto-detect)
// scope:         string  (opcional, default "project")
// topic_key:     string  (opcional)
// session_id se resuelve del contexto MCP (patrón handleSave línea 741)
```

## Testing Strategy

| Layer | What to Test | Approach |
|-------|-------------|----------|
| Unit | Atomicidad: si INSERT falla, los deletes se revierten | Mock exec para forzar error en INSERT. Assert que obs originales siguen sin `deleted_at`. |
| Unit | Soft-delete correcto | Consolidar 3 obs. Assert `deleted_at IS NOT NULL` en las 3. Assert nueva obs existe con `sync_id`. |
| Unit | Sync mutations encoladas | Consolidar 3→1. Assert 4 mutations: 3 `SyncOpDelete` + 1 `SyncOpUpsert`. |
| Unit | IDs inexistentes ignorados | Pasar `[999999]` en `ids_to_delete`. Assert no error y la nueva obs se crea igual. |

## Migration / Rollout

No migration required. Solo añade funcionalidad nueva. Backward-compatible.

## Open Questions

Ninguna.
