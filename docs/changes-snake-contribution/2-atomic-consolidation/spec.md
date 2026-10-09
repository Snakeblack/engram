# Spec: Consolidación Atómica

## Overview
Añadir herramienta MCP `mem_consolidate` permitiendo consolidar múltiples memorias en una, respetando el pipeline de sincronización.

## Requirements

### Parámetros MCP
| Parámetro | Tipo | Requerido | Descripción |
|-----------|------|-----------|-------------|
| `ids_to_delete` | array of integers | Sí | IDs de observations a soft-deletear. |
| `new_title` | string | Sí | Título de la nueva memoria consolidada. |
| `new_content` | string | Sí | Contenido de la nueva memoria consolidada. |
| `type` | string | No | Tipo de la nueva memoria (default: `"manual"`). |
| `project` | string | No | Proyecto (auto-resuelto del contexto si vacío). |
| `scope` | string | No | Scope (default: `"project"`). |
| `topic_key` | string | No | Topic key para la nueva memoria. |

### Comportamiento
1. La operación DEBE ser atómica (una sola transacción SQLite via `s.withTx()`).
2. Cada ID en `ids_to_delete` se soft-deletea: `UPDATE observations SET deleted_at = datetime('now'), updated_at = datetime('now') WHERE id = ? AND deleted_at IS NULL`.
3. Cada soft-delete DEBE encolar una sync mutation: `enqueueSyncMutationTx(tx, SyncEntityObservation, obs.SyncID, SyncOpDelete, ...)`.
4. IDs que no existen o ya están borrados se ignoran silenciosamente.
5. La nueva observación se inserta con `newSyncID("obs")` y encola su propia sync mutation: `enqueueSyncMutationTx(tx, SyncEntityObservation, syncID, SyncOpUpsert, ...)`.
6. El `session_id` se resuelve del contexto MCP actual (como hace `handleSave`).
7. La herramienta se registra en `ProfileAgent`.
8. Si la inserción falla, la transacción entera se revierte (ningún borrado se aplica).

## Implementation Details

### Store (`store.go`)
```go
func (s *Store) Consolidate(idsToDelete []int64, p AddObservationParams) (int64, error) {
    // Usa s.withTx() para atomicidad
    // Para cada id: getObservationTx → soft delete → enqueueSyncMutationTx(SyncOpDelete)
    // Luego: INSERT nueva obs con newSyncID("obs") → enqueueSyncMutationTx(SyncOpUpsert)
}
```

### MCP (`mcp.go`)
1. Registrar `mem_consolidate` en `ProfileAgent`.
2. Handler `handleConsolidate` parsea JSON args → llama `store.Consolidate()`.

## Test Criteria
- Crear 3 memorias. Consolidar en 1. Verificar que las 3 originales tienen `deleted_at` y la nueva aparece en búsqueda.
- Verificar que se encolaron 4 sync mutations (3 deletes + 1 upsert).
- Intentar consolidar con un ID inexistente: la operación completa igualmente.
