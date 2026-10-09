# Proposal: Consolidación Atómica (Garbage Collection de Contexto)

## Intent
Prevenir la acumulación de ruido y contexto obsoleto dando al agente una vía segura para consolidar múltiples memorias antiguas en una sola, respetando el pipeline de sincronización.

## Scope
### In Scope
- Creación del tool `mem_consolidate` en `mcp.go`.
- Creación de función `Consolidate()` en `store.go` usando `s.withTx()`.
- Encolado de sync mutations (delete por cada ID + upsert de la nueva memoria).

### Out of Scope
- Consolidación automática (background job). Siempre guiada por el agente.
- Hard delete. Se usará soft delete (`deleted_at`) para preservar auditabilidad y sync.

## Capabilities
### New Capabilities
- `mem_consolidate`: Soft-deletes un array de IDs e inserta una nueva memoria de forma atómica, encolando sync mutations.

## Approach
Implementar usando `s.withTx()` en `store.go`:
1. Para cada ID en `ids_to_delete`: soft-delete (`UPDATE ... SET deleted_at`) + `enqueueSyncMutationTx(SyncOpDelete)`.
2. INSERT nueva observación con `newSyncID("obs")` + `enqueueSyncMutationTx(SyncOpUpsert)`.
3. Todo en una sola transacción SQLite.

## Affected Areas
| Area | Impact | Description |
|------|--------|-------------|
| `internal/store/store.go` | New | Función `Consolidate()` con transacción atómica y sync mutations. |
| `internal/mcp/mcp.go` | Modified | Tool registrada en `ProfileAgent`, handler `handleConsolidate`. |

## Risks
| Risk | Likelihood | Mitigation |
|------|------------|------------|
| IDs no existentes o ya borrados | Low | Ignorar silenciosamente (como hace `applyObservationDeleteTx`). |
| Sync mutations faltantes | Med | Seguir el patrón exacto de `DeleteObservation` para cada borrado. |

## Rollback Plan
Borrar el tool de `mcp.go`, el handler, y la función en el store.

## Dependencies
- Ninguna.

## Success Criteria
- [ ] El agente puede llamar a `mem_consolidate` exitosamente.
- [ ] Los IDs viejos se marcan como `deleted_at IS NOT NULL`.
- [ ] La nueva memoria aparece en búsquedas subsecuentes con su propio `sync_id`.
- [ ] Las sync mutations se encolan correctamente para cada operación.
