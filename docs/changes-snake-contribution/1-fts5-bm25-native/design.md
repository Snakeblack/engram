# Design: FTS5 BM25 Native

## Technical Approach

Modificación quirúrgica de 2 líneas SQL en `Store.Search` para reemplazar el ranking plano de FTS5 por `bm25()` ponderado. No se tocan interfaces, structs ni contratos MCP. La ruta directa por `topic_key` (Rank=-1000) se preserva intacta.

## Architecture Decisions

### Decision: BM25 inline vs función de ranking personalizada

| Opción | Tradeoff | Decisión |
|--------|----------|----------|
| `bm25()` inline en query | Acopla los pesos a la query; si cambian las columnas FTS, hay que actualizar | ✅ Elegida |
| `rank` column + custom function | Requiere registrar función SQL en Go; overhead innecesario | Descartada |

**Rationale**: `bm25()` es nativo de FTS5, cero overhead, y el esquema FTS5 no cambia frecuentemente. Un comentario SQL documental basta como protección.

### Decision: Pesos de columnas

| Columna | Peso | Justificación |
|---------|------|---------------|
| `title` | 5.0 | Campo más discriminante para identidad semántica |
| `content` | 1.0 | Baseline — cuerpo extenso, matches frecuentes pero ruidosos |
| `tool_name` | 0.0 | Irrelevante para búsqueda semántica |
| `type` | 0.0 | Filtrado ya cubierto por `SearchOptions.Type` |
| `project` | 0.0 | Filtrado ya cubierto por `SearchOptions.Project` |
| `topic_key` | 3.0 | Clave relacional; match aquí indica alta afinidad temática |

## Data Flow

```
Search(query, opts)
    │
    ├── [ruta topic_key] query contiene "/" → SELECT directo → Rank = -1000
    │
    └── [ruta FTS5] → MATCH con bm25(observations_fts, 5,1,0,0,0,3) as rank
                       ORDER BY rank
    │
    └── Merge + dedup por ID → []SearchResult
```

Sin cambios en el flujo. Solo cambia cómo SQLite calcula `rank` internamente.

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `internal/store/store.go:2516` | Modify | `fts.rank` → `bm25(observations_fts, 5.0, 1.0, 0.0, 0.0, 0.0, 3.0) as rank` |
| `internal/store/store.go:2538` | Modify | `ORDER BY fts.rank` → `ORDER BY rank` |

## Interfaces / Contracts

Sin cambios. `SearchResult.Rank` (`float64`) ya se escanea en línea 2560 y el alias `rank` se mantiene.

## Testing Strategy

| Layer | What to Test | Approach |
|-------|-------------|----------|
| Unit | Ranking order cambia con BM25 | Insertar 3 obs con mismo término en `title`, `content`, `topic_key`. Buscar término. Assert orden: title > topic_key > content. |
| Unit | Sin regresión en ruta topic_key | Buscar con query conteniendo `/`. Assert Rank=-1000 intacto. |
| Unit | Sin errores SQL | Ejecutar `Search` con queries variados. Assert no panic ni error. |

## Migration / Rollout

No migration required. Cambio de comportamiento de ranking — no hay datos que migrar.

## Open Questions

Ninguna.
