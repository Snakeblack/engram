# Tasks: Graph-Linking Implícito

## Phase 1: TDD — Store Tests First (RED)

- [ ] 1.1 **RED**: En `internal/store/store_test.go`, crear `TestRelatedByTopicKeysReturnsSiblings`. Insertar 3 obs con `topic_key = "auth/jwt"`. Llamar `RelatedByTopicKeys(["auth/jwt"], "proj", "project", [id1])`. Assert: retorna 2 obs (excluye `id1`). **DEBE FALLAR** (función no existe).
- [ ] 1.2 **RED**: Crear `TestRelatedByTopicKeysExcludesDeleted`. Insertar 2 obs con mismo topic_key, soft-deletear una. Assert: solo retorna 1. **DEBE FALLAR**.
- [ ] 1.3 **RED**: Crear `TestRelatedByTopicKeysRespectsProjectScope`. Insertar obs con mismo topic_key pero distinto project. Assert: no retorna obs de otro project. **DEBE FALLAR**.
- [ ] 1.4 **RED**: Crear `TestRelatedByTopicKeysLimitsTo5PerKey`. Insertar 10 obs con mismo topic_key. Assert: retorna máximo 5 por key. **DEBE FALLAR**.
- [ ] 1.5 **RED**: Crear `TestRelatedByTopicKeysEmptyKeysNoQuery`. Llamar con `keys = []`. Assert: retorna mapa vacío, sin error. **DEBE FALLAR** (o podría pasar si función devuelve mapa vacío por defecto).

## Phase 2: Store Implementation (GREEN)

- [ ] 2.1 **GREEN**: En `internal/store/store.go`, definir `type RelatedObservation struct { ID int64; Title string }`.
- [ ] 2.2 **GREEN**: Implementar `func (s *Store) RelatedByTopicKeys(keys []string, project, scope string, excludeIDs []int64) (map[string][]RelatedObservation, error)`. Query: `SELECT id, title, topic_key FROM observations WHERE topic_key IN (?) AND deleted_at IS NULL AND project = ? AND scope = ? AND id NOT IN (?) ORDER BY updated_at DESC`. Agrupar por topic_key con LIMIT 5 por key.
- [ ] 2.3 **GREEN**: Ejecutar `go test ./internal/store/ -run "TestRelatedByTopicKeys"` — los 5 tests DEBEN PASAR.

## Phase 3: TDD — MCP Integration Test (RED)

- [ ] 3.1 **RED**: En `internal/mcp/mcp_test.go`, crear `TestSearchIncludesRelatedMetadata`. Insertar obs con topic_keys, invocar `mem_search`. Assert: la respuesta contiene texto `Related (topic` con IDs y títulos de hermanos. **DEBE FALLAR** (handleSearch no inyecta metadata aún).
- [ ] 3.2 **RED**: Crear `TestSearchNoRelatedWhenNoTopicKeys`. Insertar obs sin topic_key. Buscar. Assert: respuesta NO contiene "Related". **DEBE FALLAR** (o podría pasar si no se inyecta nada).

## Phase 4: MCP Integration (GREEN)

- [ ] 4.1 **GREEN**: En `internal/mcp/mcp.go`, en `handleSearch` (~línea 725), tras el loop de resultados: recolectar topic_keys no vacíos, llamar `store.RelatedByTopicKeys()`, serializar como `Related (topic "key"): ID X (Title), ID Y (Title)` al final de la respuesta.
- [ ] 4.2 **GREEN**: Ejecutar `go test ./internal/mcp/ -run "TestSearch"` — los 2 tests de Phase 3 DEBEN PASAR.

## Phase 5: Refactor + Verificación Full

- [ ] 5.1 **REFACTOR**: Ejecutar `go test ./internal/store/` completo — sin regresiones.
- [ ] 5.2 **REFACTOR**: Ejecutar `go test ./internal/mcp/` completo — sin regresiones.
- [ ] 5.3 **REFACTOR**: Verificar que la serialización de related no rompe el formato existente de respuesta de `handleSearch`.
