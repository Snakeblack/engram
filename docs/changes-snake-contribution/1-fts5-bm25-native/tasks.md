# Tasks: FTS5 BM25 Native

## Phase 1: TDD — Tests First (RED)

- [ ] 1.1 **RED**: En `internal/store/store_test.go`, crear `TestBM25RankingOrdersTitleAboveTopicKeyAboveContent`. Insertar 3 obs con la palabra "kubernetes" en `title`, `topic_key` y `content` respectivamente. Buscar "kubernetes". Assert orden: title-match primero, topic_key-match segundo, content-match tercero. **Este test DEBE FALLAR** con el ranking actual.
- [ ] 1.2 **RED**: Crear `TestBM25DoesNotAffectTopicKeyDirectRoute`. Insertar obs con `topic_key = "auth/jwt"`. Buscar "auth/jwt" (contiene `/`). Assert `Rank == -1000`. **Este test DEBE PASAR** ya (es regresión guard).
- [ ] 1.3 **RED**: Crear `TestBM25SearchReturnsNoSQLErrors`. Ejecutar `Search` con queries vacíos, con caracteres especiales, y con queries muy largos. Assert no error en ninguno.

## Phase 2: Implementation (GREEN)

- [ ] 2.1 **GREEN**: En `internal/store/store.go:~2516`, reemplazar `fts.rank` por `bm25(observations_fts, 5.0, 1.0, 0.0, 0.0, 0.0, 3.0) as rank`. Añadir comentario SQL: `-- weights: title=5, content=1, tool_name=0, type=0, project=0, topic_key=3`.
- [ ] 2.2 **GREEN**: En `internal/store/store.go:~2538`, reemplazar `ORDER BY fts.rank` por `ORDER BY rank`. Ejecutar los 3 tests de Phase 1: todos deben PASAR.

## Phase 3: Refactor + Verificación

- [ ] 3.1 **REFACTOR**: Verificar que `go test ./internal/store/ -run "TestBM25"` pasa los 3 tests.
- [ ] 3.2 **REFACTOR**: Ejecutar `go test ./internal/store/` completo para verificar que no hay regresiones en los tests existentes de Search.
