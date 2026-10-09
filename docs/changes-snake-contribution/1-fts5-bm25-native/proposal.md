# Proposal: Implementación de BM25 Nativo (Optimización de FTS5)

## Intent
Mejorar la relevancia de la búsqueda de memoria mediante el uso de pesos BM25, para que los matches en `title` y `topic_key` tengan prioridad sobre el `content` o metadatos.

## Scope
### In Scope
- Actualización de la query FTS5 en `Search` (SELECT y ORDER BY).
### Out of Scope
- Migración a bases vectoriales.
- Modificación de la ruta directa por `topic_key` (ya tiene prioridad absoluta con Rank -1000).

## Capabilities
### Modified Capabilities
- `mem_search`: Modificación en la relevancia de resultados sin cambiar el contrato de la API.

## Approach
En `store.go`, función `Search`:
1. Reemplazar `fts.rank` en el SELECT por `bm25(observations_fts, 5.0, 1.0, 0.0, 0.0, 0.0, 3.0) as rank`.
2. Cambiar `ORDER BY fts.rank` a `ORDER BY rank`.
3. El `rows.Scan(&sr.Rank)` existente seguirá funcionando porque el alias `rank` se mantiene.

## Affected Areas
| Area | Impact | Description |
|------|--------|-------------|
| `internal/store/store.go:2516` | Modified | Columna rank en SELECT cambia a bm25(). |
| `internal/store/store.go:2538` | Modified | ORDER BY cambia de `fts.rank` a `rank`. |

## Risks
| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Syntax Error en SQLite | Low | Los 6 pesos coinciden con las 6 columnas de `observations_fts`. |
| Cambio futuro en columnas FTS | Low | Documentar que agregar columnas a la FTS requiere actualizar los pesos. |

## Rollback Plan
Revertir `bm25(...)` a `fts.rank` en las dos líneas modificadas.

## Dependencies
- Ninguna.

## Success Criteria
- [ ] La búsqueda retorna primero los resultados donde el match está en `title` o `topic_key`.
- [ ] No hay errores de sintaxis SQLite en la query.
