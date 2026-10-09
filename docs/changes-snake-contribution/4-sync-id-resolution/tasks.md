# Tasks: Sync ID Resolution & LWW

## Phase 1: TDD — Idempotency Tests (RED)

- [ ] 1.1 **RED**: En `internal/store/store_test.go`, crear `TestImportLWWIdempotentReimport`. Exportar store → reimportar el mismo dump. Assert: `ObservationsInserted=0, ObservationsSkipped=N, PromptsSkipped=M`. **DEBE FALLAR** (Import actual duplica filas).
- [ ] 1.2 **RED**: Crear `TestImportLWWUpdatesNewerObservation`. Exportar → avanzar `updated_at` de una obs en el dump → reimportar. Assert: `ObservationsUpdated=1`. **DEBE FALLAR**.
- [ ] 1.3 **RED**: Crear `TestImportLWWSkipsOlderObservation`. Exportar → avanzar local `updated_at` → reimportar dump original. Assert: `ObservationsSkipped=N`, contenido local intacto. **DEBE FALLAR**.
- [ ] 1.4 **RED**: Crear `TestImportLWWPromptsAreImmutable`. Exportar → modificar contenido del prompt en dump → reimportar. Assert: `PromptsSkipped=1`, contenido local inalterado. **DEBE FALLAR**.

## Phase 2: TDD — deleted_at Tests (RED)

- [ ] 2.1 **RED**: Crear `TestImportLWWPreservesDeletedAtOnInsert`. Construir dump con obs que tiene `deleted_at NOT NULL`. Importar en store vacío. Assert: la obs en DB tiene `deleted_at` preservado. **DEBE FALLAR**.
- [ ] 2.2 **RED**: Crear `TestImportLWWSoftDeletePropagatesViaLWW`. Local activa + payload soft-deleted con `updated_at` más reciente. Import. Assert: local queda con `deleted_at NOT NULL`. **DEBE FALLAR**.
- [ ] 2.3 **RED**: Crear `TestImportLWWRestorationViaLWW`. Local soft-deleted + payload activa con `updated_at` más reciente. Import. Assert: local queda con `deleted_at IS NULL`. **DEBE FALLAR**.

## Phase 3: TDD — ImportResult Struct (RED)

- [ ] 3.1 **RED**: Crear `TestImportResultHasGranularCounters`. Llamar `Import()` y verificar que el resultado tiene campos `ObservationsInserted`, `ObservationsUpdated`, `ObservationsSkipped`, `PromptsInserted`, `PromptsSkipped`. **DEBE FALLAR** (struct actual no tiene esos campos).

## Phase 4: Implementation (GREEN)

- [ ] 4.1 **GREEN**: En `internal/store/store.go:~2884`, expandir `ImportResult` con los 6 campos granulares: `SessionsImported`, `ObservationsInserted`, `ObservationsUpdated`, `ObservationsSkipped`, `PromptsInserted`, `PromptsSkipped`.
- [ ] 4.2 **GREEN**: En `internal/store/store.go:~2836-2862`, reemplazar INSERT ciego de observations por bucle LWW: `SELECT id, updated_at, deleted_at WHERE sync_id = ? LIMIT 1` → ErrNoRows=INSERT / Payload>Local=UPDATE / else=SKIP. El UPDATE incluye TODOS los campos incluyendo `deleted_at`.
- [ ] 4.3 **GREEN**: En `internal/store/store.go:~2864-2875`, reemplazar INSERT ciego de prompts por bucle SKIP: `SELECT id WHERE sync_id = ? LIMIT 1` → ErrNoRows=INSERT / Exists=SKIP.
- [ ] 4.4 **GREEN**: Actualizar contadores en `ImportResult` durante cada iteración del bucle.
- [ ] 4.5 **GREEN**: Ejecutar `go test ./internal/store/ -run "TestImportLWW"` — los 8 tests de Phases 1-3 DEBEN PASAR.

## Phase 5: TDD — newSyncID Normalization (RED + GREEN)

- [ ] 5.1 **RED**: Crear `TestNewSyncIDFormatConsistency`. Llamar `newSyncID("obs")` 100 veces. Assert: todos matchean regex `^obs-[0-9a-f]{16}$`. Verificar longitud fija. **Puede PASAR** si formato actual ya es correcto — es regression guard.
- [ ] 5.2 **GREEN**: Si 5.1 falla, normalizar `newSyncID()` en `store.go:~5302` a formato `{prefix}-{16hex}` consistente. Añadir comentario documentando la decisión.

## Phase 6: Refactor + Verificación Full

- [ ] 6.1 **REFACTOR**: Ejecutar `go test ./internal/store/` completo — sin regresiones (especialmente `TestExportProjectPreservesSessionReferentialClosure` que ya usa `Import()`).
- [ ] 6.2 **REFACTOR**: Verificar que callers de `Import()` (como CLI import) manejan correctamente el nuevo `ImportResult` expandido.
- [ ] 6.3 **REFACTOR**: Verificar que tests existentes de Export/Import no se rompen con los contadores nuevos.
