# Tasks: Consolidación Atómica

## Phase 1: TDD — Store Tests First (RED)

- [ ] 1.1 **RED**: En `internal/store/store_test.go`, crear `TestConsolidateDeletesSourcesAndCreatesNewObservation`. Crear 3 obs, llamar `Consolidate(ids, params)`. Assert: las 3 originales tienen `deleted_at IS NOT NULL`, la nueva existe con `sync_id` válido. **DEBE FALLAR** (función no existe).
- [ ] 1.2 **RED**: Crear `TestConsolidateIsAtomic`. Forzar escenario donde la nueva obs fallará (ej: `session_id` vacío si hay NOT NULL). Assert: las obs originales NO tienen `deleted_at` (rollback). **DEBE FALLAR**.
- [ ] 1.3 **RED**: Crear `TestConsolidateEnqueuesSyncMutations`. Consolidar 3→1. Consultar `sync_mutations` table. Assert: 3 mutations `delete` + 1 mutation `upsert`. **DEBE FALLAR**.
- [ ] 1.4 **RED**: Crear `TestConsolidateIgnoresNonExistentIDs`. Llamar `Consolidate([999999], params)`. Assert: no error, nueva obs se crea igualmente. **DEBE FALLAR**.

## Phase 2: Store Implementation (GREEN)

- [ ] 2.1 **GREEN**: En `internal/store/store.go`, implementar `func (s *Store) Consolidate(idsToDelete []int64, p AddObservationParams) (int64, error)` usando `s.withTx()`. Loop: `getObservationTx(id)` → soft-delete → `enqueueSyncMutationTx(SyncOpDelete)`. Luego INSERT nueva obs con `newSyncID("obs")` → `enqueueSyncMutationTx(SyncOpUpsert)`.
- [ ] 2.2 **GREEN**: Ejecutar `go test ./internal/store/ -run "TestConsolidate"` — los 4 tests de Phase 1 DEBEN PASAR.

## Phase 3: TDD — MCP Handler Tests (RED)

- [ ] 3.1 **RED**: En `internal/mcp/mcp_test.go`, crear `TestHandleConsolidateToolExists`. Verificar que `mem_consolidate` aparece como tool disponible en `ProfileAgent`. **DEBE FALLAR** (tool no registrado).
- [ ] 3.2 **RED**: Crear `TestHandleConsolidateCallSucceeds`. Invocar `mem_consolidate` con args JSON válidos. Assert: respuesta exitosa con ID de nueva obs. **DEBE FALLAR**.

## Phase 4: MCP Implementation (GREEN)

- [ ] 4.1 **GREEN**: En `internal/mcp/mcp.go`, registrar `mem_consolidate` en `ProfileAgent` (zona `~línea 70`). Definir parámetros: `ids_to_delete` (requerido), `new_title` (requerido), `new_content` (requerido), `type`, `project`, `scope`, `topic_key`.
- [ ] 4.2 **GREEN**: Implementar `handleConsolidate()` siguiendo el patrón de `handleSave` (línea 741): parsear args JSON, resolver `session_id` del contexto, construir `AddObservationParams`, llamar `store.Consolidate()`.
- [ ] 4.3 **GREEN**: Ejecutar `go test ./internal/mcp/ -run "TestHandleConsolidate"` — los 2 tests de Phase 3 DEBEN PASAR.

## Phase 5: Refactor + Verificación Full

- [ ] 5.1 **REFACTOR**: Ejecutar `go test ./internal/store/` completo — sin regresiones.
- [ ] 5.2 **REFACTOR**: Ejecutar `go test ./internal/mcp/` completo — sin regresiones.
- [ ] 5.3 **REFACTOR**: Revisar que `handleConsolidate` no duplica lógica de parseo ya existente en helpers comunes.
