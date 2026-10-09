# Spec: FTS5 BM25 Native

## Overview
Reemplazar el ranking predeterminado de FTS5 por un cálculo BM25 ponderado.

## Requirements
- Los pesos deben coincidir 1:1 con las 6 columnas de `observations_fts`:
  - `title` = 5.0
  - `content` = 1.0
  - `tool_name` = 0.0
  - `type` = 0.0
  - `project` = 0.0
  - `topic_key` = 3.0
- El alias de la columna debe ser `rank` para mantener compatibilidad con `rows.Scan(&sr.Rank)`.
- La ruta directa por `topic_key` (Rank = -1000) NO debe modificarse.

## Implementation Details
Dos cambios en `store.go`, función `Search`:

```sql
-- Línea 2516: SELECT column
bm25(observations_fts, 5.0, 1.0, 0.0, 0.0, 0.0, 3.0) as rank

-- Línea 2538: ORDER BY
ORDER BY rank LIMIT ?
```

## Test Criteria
- Insertar 3 observaciones: una con match en `title`, otra en `content`, otra en `topic_key`.
- Buscar un término que aparezca en las 3. El resultado con match en `title` debe aparecer primero, seguido del match en `topic_key`, luego `content`.
