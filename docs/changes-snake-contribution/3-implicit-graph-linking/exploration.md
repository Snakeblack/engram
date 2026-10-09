## Exploration: Graph-Linking Implícito

### Current State
Las búsquedas FTS5 recuperan contexto por match léxico, pero no recuperan el contexto relacional "cercano" que podría ser vital si las palabras no coinciden exactamente.

El campo `topic_key` ya tiene un índice compuesto optimizado (línea 727 de `store.go`):
```sql
CREATE INDEX idx_obs_topic ON observations(topic_key, project, scope, updated_at DESC)
```

El sistema ya usa `topic_key` extensivamente:
- Upsert por `topic_key` en `AddObservation` (líneas 1827-1868).
- Backfill de topic_keys vacíos a NULL (línea 785).
- La ruta directa de búsqueda por `topic_key` (líneas 2467-2508).

La serialización de resultados hacia MCP ocurre en `internal/mcp/mcp.go` (handler `handleSearch`), donde se formatea el texto que el LLM recibe. Es ahí donde se inyectaría la metadata relacional.

### Affected Areas
- `internal/store/store.go` — Función `Search`, query secundaria para obtener IDs hermanos por topic_key.
- `internal/mcp/mcp.go` — Handler `handleSearch`, serialización de la metadata relacional en la respuesta.

### Approaches
1. **Base de datos de grafos o embeddings vectoriales** — Viola la filosofía zero-dependencies y single binary.
   - Pros: Semántica real.
   - Cons: Dependencia externa, peso del binario.
   - Effort: High

2. **Metadata en Search vía Topic Key** — Tras recuperar resultados FTS5, recolectar topic_keys únicos y consultar IDs hermanos. Inyectar en la respuesta MCP.
   - Pros: Cero dependencias, el índice `idx_obs_topic` ya está optimizado para esto, costo CPU mínimo.
   - Cons: Solo detecta relaciones por topic_key, no por semántica real.
   - Effort: Low/Medium

### Recommendation
Opción 2. El índice compuesto cubre la query secundaria. La inyección es en texto plano, no cambia el contrato de la API.

### Risks
- Overhead de CPU por la doble consulta (mitigado por el índice compuesto).
- Si no se filtra por `deleted_at IS NULL`, se inyectarían IDs de memorias borradas.
- Si no se filtra por `project` y `scope`, se traerían topics de otros proyectos.

### Ready for Proposal
Yes
