# Proposal: Resolución de Colisiones en Sync y Primary Keys

## Intent
Prevenir corrupción de datos y duplicados al importar archivos `.en`, implementando una resolución manual de conflictos por `sync_id` (basada en Last Write Wins) en el pipeline de `Import()`, dado que el esquema actual carece de constraints `UNIQUE` para este campo.

## Scope
### In Scope
- Refactorizar `Import()` (línea 2813) para usar lógica `SELECT` + `INSERT/UPDATE` iterativa.
- Aplicar LWW basado en `updated_at` para `observations`.
- Aplicar política "SKIP si existe" para `user_prompts` (son inmutables, sin `updated_at`).
- Normalizar `newSyncID()` para generar un formato consistente.

### Out of Scope
- Migración del ID primario entero de la tabla SQL.
- Añadir constraint `UNIQUE(sync_id)` al schema (requeriría migración destructiva si hay duplicados legacy).
- Modificar la lógica de sobrescritura ciega del pipeline de sync mutations (`applyObservationUpsertTx`).

## Capabilities
### Modified Capabilities
- `Import()`: Resolución inteligente de conflictos en lugar de inserción masiva a ciegas.

## Approach
1. **Normalizar `newSyncID`**: Elegir formato único (`obs-{hex}`).
2. **Refactorizar `Import()` para Observations**:
   - `SELECT id, updated_at FROM observations WHERE sync_id = ?`.
   - Si no existe: INSERT.
   - Si existe y `updated_at` entrante > local: UPDATE.
   - Si existe y `updated_at` entrante <= local: SKIP.
3. **Refactorizar `Import()` para Prompts**:
   - `SELECT id FROM user_prompts WHERE sync_id = ?`.
   - Si no existe: INSERT.
   - Si existe: SKIP (son inmutables).
4. **Expandir Reporte**: Actualizar `ImportResult` para contar inserted, updated, skipped.

## Affected Areas
| Area | Impact | Description |
|------|--------|-------------|
| `internal/store/store.go:2813` | Modified | `Import()` → Resolución manual LWW para obs y prompts. |
| `internal/store/store.go:5302` | Modified | `newSyncID()` → Formato normalizado. |
| `internal/store/store.go:797` | Modified | Backfill SQL → Alinear formato. |

## Risks
| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Pérdida de datos locales en conflicto | Med | LWW puro asegura que el contenido más nuevo siempre sobrevive. |
| Overhead de N queries SELECT | Low | Las importaciones manuales no son operaciones de alta frecuencia. |
| Sin constraint `UNIQUE` | Confirmed | El `SELECT` manual mitiga esto de forma segura. |

## Rollback Plan
Revertir `Import()` a la lógica de inserción masiva original.

## Dependencies
- Todos los registros entrantes de exportaciones recientes ya tendrán `sync_id`.
- Registros antiguos (de exportaciones viejas) podrían requerir fallback si carecen de `sync_id` (se insertarán y obtendrán uno nuevo).

## Success Criteria
- [ ] Importar un mismo archivo `.en` dos veces no genera filas duplicadas.
- [ ] Registros más recientes en el archivo sobrescriben a los viejos locales.
- [ ] Registros más recientes locales sobreviven a importaciones de archivos antiguos.
- [ ] El objeto JSON `ImportResult` muestra métricas granulares de la resolución.
