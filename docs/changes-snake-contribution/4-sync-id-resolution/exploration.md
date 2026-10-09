## Exploration: Resolución de Colisiones en Sync y Primary Keys

### Current State
El sistema tiene **dos pipelines de importación/sync de datos**:

1. **Pipeline legacy `Import()`** (línea 2813 de `store.go`): Hace `INSERT INTO observations` sin UPSERT. Genera nuevos IDs autoincrement. Se usa para importar dumps completos (archivos `.en`). **No maneja conflictos de sync_id, lo que genera duplicados al reimportar.**

2. **Pipeline de sync mutations** (líneas 4634-4734): Usa `applyObservationUpsertTx()`. Busca por `sync_id` via `getObservationBySyncIDTx()` y hace UPDATE si existe o INSERT si no. Sin embargo, **este pipeline sobrescribe ciegamente el registro local**, asumiendo que el remoto (cloud) siempre es la fuente de verdad. **No implementa Last Write Wins (LWW).**

El esquema de SQLite **NO tiene una restricción `UNIQUE` para `sync_id`** (líneas 581-600), solo un índice. Esto significa que no se puede usar `ON CONFLICT(sync_id)` directamente; se requiere lógica manual de `SELECT` + `INSERT/UPDATE`.

Además, los prompts (`user_prompts`) importados tampoco manejan conflictos y **no tienen columna `updated_at`**, lo que significa que son inmutables.

El `sync_id` ya se genera en dos formatos distintos en el código base:
- **En Go** (`newSyncID`, línea 5302): `obs-{16 hex chars}`.
- **En backfill SQL** (línea 797): `obs-{32 hex chars}`.

### Affected Areas
- `internal/store/store.go` — Función `Import()` legacy, `newSyncID()`, y la importación de `user_prompts`.

### Approaches
1. **Pivote Semántico a `sync_id` con LWW (Recomendado)** — 
   - Normalizar el formato de `sync_id`.
   - Mejorar `Import()` para usar un loop manual de resolución: `SELECT` por `sync_id`.
   - Para `observations`: Aplicar LWW usando `updated_at`. Si existe y el local es más nuevo, saltar. Si el entrante es más nuevo, actualizar.
   - Para `user_prompts`: Al ser inmutables (sin `updated_at`), si existe el `sync_id`, saltar.
   - Pros: No rompe el schema (no requiere añadir `UNIQUE` a una tabla con posibles duplicados), resuelve el problema real del usuario al importar backups.
   - Cons: Código procedural en Go en vez de constraints SQL puro.
   - Effort: Medium

### Recommendation
Opción 1. Añadir LWW como una lógica explícita en `Import()` es crucial porque los backups importados pueden ser antiguos, a diferencia de las sync mutations remotas que se asumen como la frontera de estado.

### Risks
- Dos formatos de `sync_id` coexistentes. Aceptaremos ambos como válidos hacia atrás.
- Añadir LWW al `Import()` es un cambio de paradigma (antes el import siempre "agregaba").
- Los `user_prompts` no tienen `updated_at`, por lo que su resolución debe ser un simple `SKIP` si existen.

### Ready for Proposal
Yes
