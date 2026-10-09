## Exploration: Implementación de BM25 Nativo (Optimización de FTS5)

### Current State
El ordenamiento actual en FTS5 asume pesos idénticos (1.0) para todas las columnas de la tabla virtual `observations_fts` (title, content, tool_name, type, project, topic_key), lo que resta relevancia semántica.

La función `Search` en `store.go` tiene **dos rutas de búsqueda**:
1. **Ruta directa por `topic_key`** (líneas 2467-2508): Si el query contiene `/`, busca por `WHERE topic_key = ?` sin FTS5, asignando `Rank = -1000` (prioridad absoluta). Esta ruta NO se ve afectada por el cambio.
2. **Ruta FTS5** (líneas 2513-2538): Usa `observations_fts MATCH ?` con `ORDER BY fts.rank`. Esta es la ruta que se modifica.

Los resultados de ambas rutas se combinan en un slice único, deduplicando por ID (líneas 2547-2566).

### Affected Areas
- `internal/store/store.go` — Función `Store.Search`, query FTS5 (líneas 2513-2538).
  - Columna `fts.rank` en el SELECT (línea 2516) → cambia a `bm25(observations_fts, ...) as rank`.
  - `ORDER BY fts.rank` (línea 2538) → cambia a `ORDER BY rank`.
  - El `rows.Scan` (línea 2560) ya escanea `&sr.Rank` — el alias `rank` debe coincidir.

### Approaches
1. **BM25 nativo con pesos** — Usar `bm25(observations_fts, 5.0, 1.0, 0.0, 0.0, 0.0, 3.0)` directamente en la query FTS5.
   - Pros: Cero dependencias, ejecución rápida, resuelve el problema.
   - Cons: Sintaxis estricta (requiere los 6 pesos exactos). Si se agregan columnas a la FTS en el futuro, los pesos deben actualizarse.
   - Effort: Low

### Recommendation
Implementar el enfoque 1: BM25 nativo, asignando peso 5.0 a `title` y 3.0 a `topic_key`.

### Risks
- Error de sintaxis SQL si la cantidad de columnas virtuales cambia y no se actualizan los pesos.
- No afecta la ruta directa por `topic_key` (que ya tiene prioridad absoluta con Rank -1000).

### Ready for Proposal
Yes
