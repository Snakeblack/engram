# Design: Graph-Linking Implícito

## Technical Approach

Post-procesamiento en dos capas: (1) query secundaria en `store.go` para recolectar hermanos por `topic_key` y (2) inyección de metadata relacional como texto en `handleSearch` de `mcp.go`. No se modifica el contrato de la API MCP — la metadata va como texto adicional en la respuesta existente.

## Architecture Decisions

### Decision: Función separada vs inline en Search

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| Función separada `RelatedByTopicKeys()` en store | Testeable en aislamiento, reutilizable | ✅ Elegida |
| Código inline dentro de `Search()` | Menos indirección pero mezcla responsabilidades | Descartada |

**Rationale**: `Search()` ya tiene dos rutas (directa + FTS5) y 100+ líneas. Añadir más lógica inline haría la función inmantenible. Una función separada se puede testear sin montar todo el flujo de búsqueda.

### Decision: Inyección en store vs en handler MCP

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| Inyectar en `handleSearch` (mcp.go) | La serialización a texto ya ocurre ahí; el store retorna datos puros | ✅ Elegida |
| Retornar metadata embebida en `SearchResult` | Contamina la struct del store con concerns de presentación | Descartada |

**Rationale**: El store debe retornar datos. El handler MCP es la capa de presentación. Separación de responsabilidades.

### Decision: Límite de hermanos por topic_key

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| Max 5 hermanos | Balance entre contexto y token budget | ✅ Elegida |
| Sin límite | Riesgo de inflar el contexto del LLM con docenas de IDs | Descartada |
| Max 3 hermanos | Demasiado restrictivo para topics densos | Descartada |

## Data Flow

```
handleSearch(query, opts)
    │
    └── store.Search(query, opts) → []SearchResult
            │
            ├── Recolectar topic_keys no vacíos de resultados
            │
            └── store.RelatedByTopicKeys(keys, project, scope, excludeIDs)
                    │   Usa idx_obs_topic(topic_key, project, scope, updated_at)
                    │
                    └── map[string][]RelatedObservation
            │
            └── Serializar respuesta texto:
                  "[1] #4 (decision) — JWT Middleware ..."
                  "Related (topic \"auth/jwt\"): ID 12 (Token Rotation), ID 7 (Session Fix)"
```

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `internal/store/store.go` | Create func | `RelatedByTopicKeys(keys []string, project, scope string, excludeIDs []int64) (map[string][]RelatedObservation, error)` |
| `internal/store/store.go` | Create type | `RelatedObservation{ID int64; Title string}` |
| `internal/mcp/mcp.go:725` | Modify | En `handleSearch`, tras el loop de resultados, llamar a `RelatedByTopicKeys` e inyectar texto. |

## Interfaces / Contracts

```go
// store.go — nuevo tipo y función
type RelatedObservation struct {
    ID    int64
    Title string
}

func (s *Store) RelatedByTopicKeys(
    keys []string,
    project, scope string,
    excludeIDs []int64,
) (map[string][]RelatedObservation, error)
// Retorna: mapa topic_key → slice de hermanos (max 5 por key).
// Query: SELECT id, title FROM observations
//        WHERE topic_key IN (?) AND deleted_at IS NULL
//              AND project = ? AND scope = ?
//              AND id NOT IN (excludeIDs)
//        ORDER BY updated_at DESC
```

```
// Formato de inyección en handleSearch (texto plano):
Related (topic "auth/jwt"): ID 4 (JWT Middleware), ID 12 (Token Rotation Bug)
Related (topic "infra/deploy"): ID 8 (CI Pipeline Fix)
```

## Testing Strategy

| Layer | What to Test | Approach |
|-------|-------------|----------|
| Unit | RelatedByTopicKeys retorna hermanos correctos | Insertar 3 obs con mismo topic_key. Llamar con 1 excluido. Assert 2 retornados. |
| Unit | Filtro deleted_at | Insertar obs con `deleted_at`. Assert NO aparece en related. |
| Unit | Filtro cross-project | Insertar obs con distinto project. Assert NO aparece. |
| Unit | Max 5 por topic | Insertar 10 obs con mismo topic. Assert max 5 retornadas. |
| Unit | Sin topic_keys → sin query secundaria | Buscar obs sin topic_key. Assert que `RelatedByTopicKeys` no se invoca. |

## Migration / Rollout

No migration required. El índice `idx_obs_topic` ya existe (línea 727). No hay cambios de schema.

## Open Questions

Ninguna.
