# Proposal: Graph-Linking Implícito

## Intent
Otorgar conciencia relacional al agente inyectando IDs y títulos de memorias vinculadas al mismo contexto sin requerir arquitecturas externas.

## Scope
### In Scope
- Query secundaria en `store.go:Search` para recolectar IDs y títulos hermanos por `topic_key`.
- Inyección de metadata relacional en la respuesta MCP de `handleSearch` en `mcp.go`.

### Out of Scope
- Grafos visuales interactivos o relaciones explícitas de múltiples tipos.
- Modificación del contrato JSON de la API (se inyecta como texto adicional).

## Capabilities
### Modified Capabilities
- `mem_search`: Retorno modificado para incluir metadata relacional contextual (IDs y títulos) en texto.

## Approach
1. En `store.go:Search`, tras obtener los `SearchResult`, recolectar los `topic_key` no vacíos.
2. Query secundaria: `SELECT id, title, topic_key FROM observations WHERE topic_key IN (...) AND deleted_at IS NULL AND project = ? AND scope = ?`.
3. El índice `idx_obs_topic(topic_key, project, scope, updated_at DESC)` cubre esta query.
4. En `mcp.go:handleSearch`, al serializar la respuesta, agregar un bloque de metadata por topic_key.

## Affected Areas
| Area | Impact | Description |
|------|--------|-------------|
| `internal/store/store.go` | Modified | Query secundaria en `Search` para IDs y títulos hermanos. |
| `internal/mcp/mcp.go` | Modified | Serialización de metadata relacional rica en `handleSearch`. |

## Risks
| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Búsquedas más lentas | Low | Índice compuesto ya optimizado, query limitada. |
| IDs de memorias borradas inyectadas | Med | Filtrar siempre con `AND deleted_at IS NULL`. |
| Topics de otros proyectos | Med | Filtrar por `project` y `scope` en la query secundaria. |
| Inflar el contexto del LLM | Low | Limitar a max 5 hermanos por topic y truncar títulos muy largos. |

## Rollback Plan
Revertir los cambios en `Search` y `handleSearch`. La API no cambia de contrato.

## Dependencies
- `topic_key` debe estar en el esquema y con índice (ya cumplido).
- El cambio #1 (BM25) es recomendable pero no bloqueante.

## Success Criteria
- [ ] La respuesta de `mem_search` incluye "Related (topic X): ID 1 (Setup DB), ID 2 (Fix Connection)" cuando hay hermanos.
- [ ] No se inyectan IDs de memorias con `deleted_at IS NOT NULL`.
- [ ] No se inyectan IDs de otros proyectos o scopes.
