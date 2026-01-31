# GeoSemOps Design Specification

**Semantic Operators for LLM-Powered Geospatial Data Processing**
*Version 1.0 — December 2025*

---

## 1. Executive Summary

GeoSemOps is a lightweight framework that brings LLM-powered semantic operations to geospatial data workflows. Built on DuckDB with its spatial extension, GeoSemOps provides a small, composable set of operators that enable GIS and data professionals to perform tasks that were previously difficult or impossible with traditional tools.

### 1.1 Core Value Proposition

- **GIS tooling** excels at geometry and topology (GDAL, PostGIS, DuckDB spatial)
- **LLMs** excel at understanding meaning, fuzzy matching, and classification
- **GeoSemOps** bridges this gap with:
  - Declarative semantic operations that compose naturally with SQL
  - GIS-aware primitives that understand geometry is first-class
  - Inspectable pipelines where blocking, prompts, and intermediate results are visible
  - A minimal API (four core operators) that handles 90% of use cases

### 1.2 Inspiration

- **LOTUS** (Stanford/Berkeley): Introduced the semantic operator model with `sem_map`, `sem_filter`, `sem_join`, `sem_agg`, `sem_topk`
- **DocETL** (UC Berkeley): YAML-based pipeline with `resolve` operator for entity resolution

---

## 2. Design Principles

1. **Minimal, Orthogonal Verb Set** — Four core operators that are clearly distinct
2. **One Low-Level UDF, Many Higher-Level Operators** — All operators compile to primitive LLM UDFs
3. **GIS-Aware but Not GIS-Only** — Geometry is first-class, but operators work on any table
4. **Non-Magical** — Blocking logic and prompts are visible and configurable
5. **Composable with Pure SQL** — Layer semantics on top of spatial functions

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  User API: Python wrapper or SQL table functions        │
├─────────────────────────────────────────────────────────┤
│  Semantic Operators: SEM_MAP, SEM_FILTER, SEM_MATCH,    │
│                      SEM_RESOLVE                        │
├─────────────────────────────────────────────────────────┤
│  LLM UDFs: llm_struct(), llm_bool()                     │
├─────────────────────────────────────────────────────────┤
│  DuckDB: spatial extension, vss extension               │
└─────────────────────────────────────────────────────────┘
```

### 3.1 Low-Level LLM UDFs

```sql
llm_struct(prompt TEXT, args STRUCT, schema TEXT) → STRUCT
llm_bool(prompt TEXT, args STRUCT) → BOOLEAN
```

---

## 4. Core Operators

### 4.1 SEM_MAP — Per-Row Semantic Enrichment

**Purpose:** Apply an LLM to each row to produce additional columns.

| Parameter | Description |
|-----------|-------------|
| `relation` | Input table or query |
| `columns` | Columns to pass to LLM |
| `prompt` | Jinja2 template |
| `output_schema` | DuckDB STRUCT for output |

**Example:**
```python
db.sem_map(
    relation="SELECT id, name, tags FROM pois",
    columns=["name", "tags"],
    prompt="Classify POI: {{ name }} with tags {{ tags }}",
    output_schema={"category": str, "is_chain": bool}
)
```

**Use Cases:** Category normalization, address parsing, attribute extraction

---

### 4.2 SEM_FILTER — Semantic Row Filter

**Purpose:** Keep rows where LLM returns TRUE for a natural-language condition.

| Parameter | Description |
|-----------|-------------|
| `relation` | Input table or query |
| `columns` | Columns for LLM context |
| `predicate` | Natural language condition |

**Example:**
```python
db.sem_filter(
    relation="SELECT * FROM parks",
    columns=["name", "description", "amenities"],
    predicate="This park is suitable for large public events"
)
```

---

### 4.3 SEM_MATCH — Semantic Pair Comparison

**Purpose:** Given candidate pairs, use LLM to decide if each pair is a match.

**Key Design:** Does NOT perform blocking. Candidate pairs are generated via SQL (ST_DWithin, etc.).

| Parameter | Description |
|-----------|-------------|
| `candidate_pairs` | Relation with left_*/right_* columns |
| `left_columns` | Columns from left record |
| `right_columns` | Columns from right record |
| `comparison_prompt` | Jinja2 template for comparison |

**Example:**
```sql
-- Step 1: Generate candidates with spatial blocking
CREATE TABLE candidates AS
SELECT a.*, b.*
FROM osm_pois a, city_licenses b
WHERE ST_DWithin(a.geom, b.geom, 100);

-- Step 2: Semantic matching
db.sem_match(
    candidate_pairs="candidates",
    comparison_prompt="Do these refer to the same business?"
)
```

**Output:** `is_match` (BOOLEAN), optionally `match_confidence` (FLOAT)

---

### 4.4 SEM_RESOLVE — Entity Resolution Pipeline

**Purpose:** Full pipeline: blocking → matching → clustering → canonicalization.

| Parameter | Description |
|-----------|-------------|
| `relation` | Input table |
| `blocking_keys` | Columns for embedding similarity |
| `blocking_threshold` | Min similarity (0.0-1.0) |
| `spatial_blocking` | `{geom, max_distance_m}` |
| `comparison_prompt` | Pairwise comparison prompt |
| `resolution_prompt` | Canonicalization prompt |
| `output_schema` | Schema for canonical output |

**Output Tables:**
- `entities` — One row per resolved entity
- `membership` — Maps original IDs to entity IDs

---

## 5. Comparison with LOTUS and DocETL

| Capability | GeoSemOps | LOTUS | DocETL |
|------------|-----------|-------|--------|
| Per-row enrichment | `sem_map` | `sem_map` | `map` |
| Semantic filter | `sem_filter` | `sem_filter` | `filter` |
| Pair matching | `sem_match` | `sem_join` | `equijoin` |
| Entity resolution | `sem_resolve` | — | `resolve` |
| Spatial blocking | Native | — | — |
| Geometry type | Native | — | — |

**Key Differentiators:**
- Spatial-first design with ST_DWithin blocking
- Explicit blocking (not hidden in sem_join)
- SQL-native table functions
- Smaller, focused scope

---

## 6. Implementation Roadmap

### Phase 1: LLM Plumbing + SEM_MAP (Weeks 1-2)
- [ ] Implement `llm_struct` and `llm_bool` UDFs
- [ ] Batching, rate limiting, caching
- [ ] Build `sem_map` Python wrapper
- [ ] Test on POI category normalization

### Phase 2: SEM_FILTER + SEM_MATCH (Weeks 3-4)
- [ ] Implement `sem_filter` using `llm_bool`
- [ ] Implement `sem_match` for candidate pairs
- [ ] Evaluation framework vs baselines

### Phase 3: SEM_RESOLVE (Weeks 5-7)
- [ ] Blocking logic (spatial + embedding)
- [ ] Union-find clustering
- [ ] Canonicalization via LLM
- [ ] Output entity + membership tables

### Phase 4: Polish + Evaluation (Week 8)
- [ ] Full evaluation on POI benchmark
- [ ] Documentation and tutorials
- [ ] Hello world examples

---

## 7. Open Questions

### API Surface
- Python-first vs SQL-first?
- Schema format: DuckDB STRUCT vs JSON Schema vs dataclasses?

### Blocking Strategy
- Embedding model: local (sentence-transformers) vs API?
- Blocking in SQL vs in SEM_RESOLVE?

### LLM Configuration
- Use LiteLLM for multi-provider?
- Include validation/gleaning in v1?

### Evaluation
- Benchmark datasets?
- Metrics: precision/recall + cost (tokens, latency)
- Baselines: fuzzy string + distance vs embeddings vs full LLM

---

## Appendix A: Future Operators (v2+)

- `SEM_AGG` / `SEM_REDUCE` — Semantic aggregation
- `SEM_TOPK` — Semantic ranking
- `SEM_SEARCH` — Natural language spatial search
- `SEM_ANNOTATE_REGION` — Region summaries
- Domain wrappers: `GEO_SEM_RESOLVE_POI`, etc.

---

## Appendix B: References

- LOTUS: [arXiv:2407.11418](https://arxiv.org/abs/2407.11418)
- DocETL: [ucbepic.github.io/docetl](https://ucbepic.github.io/docetl/)
- DuckDB Spatial: [duckdb.org/docs/extensions/spatial](https://duckdb.org/docs/extensions/spatial)
- Entity Resolution Survey: Binette & Steorts, "Almost All of Entity Resolution" (2022)
