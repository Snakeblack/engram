# Roadmap de Mejoras Core: Engram (Cognición y Memoria)

*Filosofía del proyecto: Single binary. Zero dependencies. Local, rápido y agnóstico. Cualquier cambio debe respetar esta base.*

*Validado contra código: `internal/store/store.go` (5508 líneas), `internal/mcp/mcp.go` (1488 líneas). Fecha: 2026-04-26.*

## 1. Implementación de BM25 Nativo (Optimización de FTS5)
**Prioridad:** 1 (Crítica / Bloqueante)
**Objetivo:** Mejorar radicalmente la relevancia de la búsqueda de SQLite FTS5 sin añadir dependencias externas ni peso al binario.

* **Validación de Código:** En `store.go` (línea 607), la tabla virtual `observations_fts` se crea con 6 columnas explícitas: `title, content, tool_name, type, project, topic_key`.
* **El Problema:** El ordenamiento por defecto de FTS5 asume pesos idénticos (1.0) para todas las columnas. Un match en el contenido irrelevante pesa igual que un match en el título.
* **Especificación Técnica:**
    * Modificar la query SQL en `Store.Search` (línea 2513 de `store.go`).
    * Reemplazar `fts.rank` (SELECT línea 2516) por `bm25(observations_fts, 5.0, 1.0, 0.0, 0.0, 0.0, 3.0) as rank`.
    * Cambiar `ORDER BY fts.rank` (línea 2538) por `ORDER BY rank`.
    * El `rows.Scan(&sr.Rank)` (línea 2560) sigue funcionando porque el alias `rank` se mantiene.
    * **Nota:** Existe una ruta directa por `topic_key` (líneas 2467-2508) que NO usa FTS5 y asigna Rank=-1000. No se modifica.
* **Documentación detallada:** [`1-fts5-bm25-native/`](1-fts5-bm25-native/)

---

## 2. Consolidación Atómica (Garbage Collection de Contexto)
**Prioridad:** 2 (Alta / Bloqueante para Agentes)
**Objetivo:** Prevenir el envenenamiento del contexto por acumulación de memorias obsoletas, otorgando al agente permisos seguros de limpieza.

* **Validación de Código:** En `mcp.go`, el perfil `ProfileAgent` (líneas 58-71) expone `mem_update` pero **NO** expone `mem_delete` (que pertenece a `ProfileAdmin`, líneas 75-80).
* **El Problema Real:** El agente no puede borrar ruido. Para consolidar 3 memorias viejas, necesitaría `mem_delete` (prohibido) + `mem_save`, consumiendo 4 rondas MCP.
* **Especificación Técnica:**
    * Crear `mem_consolidate` usando `s.withTx()` (línea 3960).
    * Soft-delete (no hard delete) de cada ID + `enqueueSyncMutationTx(SyncOpDelete)` para propagar el borrado en sync.
    * INSERT nueva observación con `newSyncID("obs")` + `enqueueSyncMutationTx(SyncOpUpsert)`.
    * Parámetros completos: `ids_to_delete`, `new_title`, `new_content`, `type`, `project`, `scope`, `topic_key`. El `session_id` se resuelve del contexto MCP.
* **Documentación detallada:** [`2-atomic-consolidation/`](2-atomic-consolidation/)

---

## 3. Graph-Linking Implícito (Relaciones referenciales nativas)
**Prioridad:** 3 (Media)
**Objetivo:** Dotar al sistema de "conciencia relacional" espacial, reemplazando la necesidad de embeddings vectoriales pesados que violan la filosofía "zero-dependencies".

* **Validación de Código:** `topic_key` tiene un índice compuesto optimizado (línea 727): `idx_obs_topic(topic_key, project, scope, updated_at DESC)`.
* **El Problema:** FTS5 recupera por léxico, pero no recupera el "vecindario" lógico si las palabras no coinciden.
* **Especificación Técnica:**
    * En `store.go:Search`, tras obtener resultados FTS5, recolectar `topic_key` no vacíos y consultar IDs hermanos.
    * Query secundaria DEBE filtrar: `deleted_at IS NULL AND project = ? AND scope = ?`.
    * La inyección de metadata ocurre en `mcp.go:handleSearch` (serialización), no en el store.
    * Formato: *"Related (topic "auth/jwt"): ID 4, ID 12"*.
* **Documentación detallada:** [`3-implicit-graph-linking/`](3-implicit-graph-linking/)

---

## 4. Resolución de Colisiones en Sync y Primary Keys (`engram sync`)
**Prioridad:** 4 (Crítica para Escalabilidad de Equipos)
**Objetivo:** Prevenir la corrupción de datos y duplicados al sincronizar bases de datos de diferentes entornos.

* **Validación de Código:** Existen **dos pipelines de import** en `store.go`:
    * **Legacy `Import()`** (línea 2813): INSERT ciego sin UPSERT. No maneja conflictos.
    * **Sync mutations** (líneas 4634-4734): `applyObservationUpsertTx()` y `applyObservationDeleteTx()` ya implementan UPSERT por `sync_id`. Este es el patrón correcto a seguir.
* **El Problema Real:** `Import()` genera duplicados porque no resuelve por `sync_id`. Además, `newSyncID()` (línea 5302: 16 hex) y el backfill SQL (línea 797: 32 hex) generan formatos distintos.
* **Especificación Técnica:**
    * Refactorizar `Import()` para hacer UPSERT por `sync_id` con LWW basado en `updated_at`.
    * Normalizar `newSyncID()` y el backfill SQL a un formato único.
    * **Descartado:** UUIDv7 (introduciría un tercer formato innecesario).
    * Reutilizar la lógica de `applyObservationUpsertTx` o extraer función compartida.
    * Expandir `ImportResult` con `inserted/updated/skipped`.
* **Documentación detallada:** [`4-sync-id-resolution/`](4-sync-id-resolution/)

---

## Patrón Transversal: Sync Mutations

Todo cambio que toque `observations` DEBE considerar `enqueueSyncMutationTx()` como parte del contrato. Este pipeline encola mutaciones para propagación entre máquinas sincronizadas. Los cambios 2, 3 y 4 interactúan directamente con él.