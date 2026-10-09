## Exploration: Consolidación Atómica (Garbage Collection de Contexto)

### Current State
El perfil `ProfileAgent` (líneas 58-71 de `mcp.go`) expone `mem_update` pero **NO** expone `mem_delete` (que pertenece a `ProfileAdmin`, líneas 75-80). Esto bloquea al agente para limpiar memorias obsoletas.

El sistema tiene un pipeline de sincronización maduro:
- Cada operación de escritura/borrado encola mutaciones via `enqueueSyncMutationTx()` (ej: `DeleteObservation`, líneas 2333-2340).
- La función `DeleteObservation` (línea 2303) soporta tanto hard delete como soft delete (`deleted_at`).
- `AddObservation` (línea 1809) genera `sync_id` via `newSyncID("obs")` y encola sync mutations automáticamente.
- El helper transaccional `s.withTx()` (línea 3960) ya está probado y disponible.

### Affected Areas
- `internal/mcp/mcp.go` — Registro del tool `mem_consolidate` en `ProfileAgent` y handler.
- `internal/store/store.go` — Nueva función transaccional `Consolidate()`.

### Approaches
1. **Agregar `mem_delete` al Agente** — Riesgoso: el LLM podría borrar contexto vital sin restricción.
   - Pros: Sin código nuevo.
   - Cons: Peligroso, pierde la garantía de atomicidad (borrar 3 + insertar 1 requeriría 4 calls MCP secuenciales).
   - Effort: Low

2. **Crear `mem_consolidate` (Atómica)** — Una sola transacción que soft-deletes IDs específicos e inserta la nueva memoria consolidada, encolando sync mutations para cada operación.
   - Pros: Atómico, seguro, una sola ronda MCP, respeta el pipeline de sync.
   - Cons: Código nuevo en store y mcp.
   - Effort: Medium

### Recommendation
Implementar la Opción 2 (`mem_consolidate`).

### Risks
- Condiciones de carrera si se borran IDs que están siendo actualizados simultáneamente (mitigado por SQLite WAL single-writer).
- Olvidar encolar sync mutations rompería la sincronización multi-máquina.

### Ready for Proposal
Yes
