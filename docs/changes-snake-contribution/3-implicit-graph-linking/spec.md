# Spec: Graph-Linking Implícito

## Overview
Inyectar un grafo relacional simulado usando `topic_key` en la respuesta de `mem_search`, incluyendo tanto el ID como el título para contexto inmediato del LLM.

## Requirements
1. Tras obtener los `SearchResult` de FTS5, recolectar los `topic_key` no vacíos de los resultados.
2. Ejecutar query secundaria:
   ```sql
   SELECT id, title, topic_key FROM observations
   WHERE topic_key IN (?, ?, ...)
     AND deleted_at IS NULL
     AND project = ?
     AND scope = ?
     AND id NOT IN (ids_ya_en_resultados)
   ORDER BY updated_at DESC
   ```
3. Limitar a un máximo razonable (ej: 5 IDs relacionados por topic_key) para no inflar la respuesta.
4. La metadata se inyecta como texto adicional en la serialización de `handleSearch` (en `mcp.go`). Formato enriquecido:
   ```
   Related (topic "architecture/auth"): ID 4 (JWT Middleware), ID 12 (Token Rotation Bug)
   ```
5. El overhead de la query secundaria no debe superar unos pocos milisegundos (cubierto por `idx_obs_topic`).
6. Si no hay topic_keys en los resultados, no se ejecuta la query secundaria.

## Implementation Details

### Store (`store.go`)
- Añadir al return de `Search` la información de hermanos, o exponer una función separada:
  `RelatedByTopicKeys(keys []string, project, scope string, excludeIDs []int64) (map[string][]RelatedObservation, error)`
  donde `RelatedObservation` contiene `ID` y `Title`.

### MCP (`mcp.go`)
- En `handleSearch`, tras obtener resultados, llamar a la función de related y concatenar la metadata rica en el texto de respuesta.

## Test Criteria
- Crear 3 memorias con `topic_key = "auth/jwt"` y 2 con `topic_key = "infra/docker"`.
- Buscar un término que matchee una de "auth/jwt". Verificar que los IDs y títulos de las otras 2 memorias "auth/jwt" aparecen como relacionadas.
- Crear una memoria con `deleted_at` y mismo topic_key. Verificar que NO aparece como relacionada.
