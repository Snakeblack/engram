# Spec: Sync ID Resolution & LWW

## Overview
Refactorizar el comando `Import()` manual para resolver colisiones por `sync_id`, aplicando Last Write Wins (LWW) a entidades mutables y saltando duplicados en entidades inmutables. 

## Requirements

### 1. Normalización de `newSyncID()` (línea 5302)
- Elegir formato único: `{prefix}-{hex}` con longitud consistente.
- Actualizar el backfill SQL (línea 797) para usar el mismo formato.

### 2. Importación de Observations (línea 2836)
Para cada observation en el payload:
```sql
-- Buscar por sync_id (ya que no hay UNIQUE constraint)
SELECT id, updated_at FROM observations WHERE sync_id = ? LIMIT 1
```
- **Si ErrNoRows:** `INSERT` usando los datos del payload. Incrementar `ObservationsInserted`.
- **Si Existe:**
  - Comparar `updated_at` (lexicográficamente o parseado).
  - Si `Payload.UpdatedAt > Local.UpdatedAt`: Ejecutar `UPDATE` con los datos del payload. Incrementar `ObservationsUpdated`.
  - Si `Payload.UpdatedAt <= Local.UpdatedAt`: No hacer nada. Incrementar `ObservationsSkipped`.

### 3. Importación de User Prompts (línea 2864)
Para cada prompt en el payload:
```sql
SELECT id FROM user_prompts WHERE sync_id = ? LIMIT 1
```
- **Si ErrNoRows:** `INSERT`. Incrementar `PromptsInserted`.
- **Si Existe:** No hacer nada (los prompts son inmutables). Incrementar `PromptsSkipped`.

### 4. Estructura de Respuesta Expandida
```go
type ImportResult struct {
    SessionsImported     int `json:"sessions_imported"`
    ObservationsInserted int `json:"observations_inserted"`
    ObservationsUpdated  int `json:"observations_updated"`
    ObservationsSkipped  int `json:"observations_skipped"`
    PromptsInserted      int `json:"prompts_inserted"`
    PromptsSkipped       int `json:"prompts_skipped"`
}
```

## Implementation Details
- Las comprobaciones `SELECT` deben ocurrir **dentro de la misma transacción** (`tx`) que los `INSERT/UPDATE` para evitar condiciones de carrera.
- Reemplazar las inserciones masivas ciegas en `Import()` por los bucles condicionales descritos arriba.

### Manejo de `deleted_at` en LWW

El `Export()` exporta **todas** las observaciones, incluyendo las que tienen `deleted_at IS NOT NULL`. El nuevo `Import()` debe tratar `deleted_at` como un campo más del payload — no asume que todo lo importado está "vivo".

Casuísticas a cubrir explícitamente:

| Estado local | Estado payload | `updated_at` ganador | Acción |
|---|---|---|---|
| Activa (`deleted_at IS NULL`) | Soft-deleted (`deleted_at NOT NULL`) | Payload > Local | `UPDATE` con `deleted_at = payload.deleted_at` → la obs queda soft-deleted localmente |
| Soft-deleted (`deleted_at NOT NULL`) | Activa (`deleted_at IS NULL`) | Payload > Local | `UPDATE` con `deleted_at = NULL` → la obs se restaura localmente |
| Activa | Soft-deleted | Local >= Payload | `SKIP` → la obs local activa sobrevive |
| Soft-deleted | Activa | Local >= Payload | `SKIP` → la obs local borrada se mantiene |

En todos los casos, el campo `deleted_at` del payload se incluye siempre en el `UPDATE`, nunca se ignora.

## Test Criteria
- Importar un dump. Re-importar el mismo dump. Verificar que no se duplican observaciones ni prompts (skipped = max).
- Modificar una observation en el dump (avanzar `updated_at`). Re-importar. Verificar que la observación local se actualiza (`ObservationsUpdated = 1`).
- Re-importar un prompt ya existente con diferente texto. Verificar que el local se mantiene intacto (`PromptsSkipped = 1`).
- Exportar una DB con una observación soft-deleted. Importar en otra instancia donde esa obs no existe. Verificar que se inserta con `deleted_at` preservado.
- Exportar una DB con una obs soft-deleted (`updated_at` reciente). Importar en instancia donde la misma obs está activa (pero más vieja). Verificar que la obs local queda soft-deleted tras el import.
