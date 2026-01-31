# GeoSemOps Intent

> The reasoning behind this library—captured so it survives beyond chat logs.

---

## The Problem

**GIS tooling is great at geometry. Terrible at meaning.**

PostGIS, GDAL, DuckDB Spatial—these tools excel at topology, projections, spatial joins. But when you need to:

- Normalize messy POI categories ("cafe" vs "coffee shop" vs "espresso bar")
- Match records across datasets with fuzzy names and addresses
- Filter features by semantic criteria ("parks suitable for concerts")
- Resolve duplicates from OSM + city data + commercial sources

...you're on your own. People write ad-hoc Python scripts, call GPT in loops, pray the prompts work.

**LLMs are great at meaning. Awkward with geometry.**

LLMs understand that "Starbucks Coffee" and "Starbucks" are the same business. They can classify, match, and reason about places. But current usage is:

- Ad-hoc (one-off scripts, not reusable)
- Not composable with SQL pipelines
- Expensive (no batching, no caching, no blocking)
- Hard to inspect (prompts buried in code)

**GeoSemOps bridges this gap.**

---

## The Vision

A library that brings semantic operations to geospatial workflows—elegantly.

### Design Principles

| Principle | What it means |
|-----------|---------------|
| **Elegant, minimal API** | Four operators handle 90% of use cases. Not a framework, a toolkit. |
| **Works out of the box** | `pip install geosemops` → immediately useful. Sensible defaults. |
| **Fits existing stacks** | DuckDB-native, SQL-composable. Not a new paradigm to learn. |
| **Inspectable** | Prompts are visible. Blocking is explicit. Intermediate results accessible. |
| **AI-agent friendly** | Clean interfaces, predictable behavior, designed to be used by agents. |

### Why DuckDB?

- Fast, embedded, no server
- Excellent spatial extension
- Growing adoption in data/geo community
- SQL is the lingua franca agents already understand

### Why not Pandas (like LOTUS)?

- SQL composes better across tools
- DuckDB handles larger-than-memory data
- Geo users already think in SQL (PostGIS heritage)
- Agents write better SQL than Pandas

---

## Inspiration

### LOTUS (Stanford/Berkeley)

Introduced semantic operators: `sem_map`, `sem_filter`, `sem_join`, `sem_agg`, `sem_topk`. Showed that LLM-powered data operations can be declarative and composable. Pandas-based.

**What we take:** The operator model. Natural language expressions ("langex") as first-class.

**What we change:** DuckDB instead of Pandas. Spatial-native. Explicit blocking.

### DocETL (UC Berkeley)

YAML pipeline system with `map`, `filter`, `reduce`, `resolve`, `equijoin`. The `resolve` operator is particularly relevant—entity resolution with comparison, clustering, canonicalization.

**What we take:** The resolve pipeline design. Test-driven operator development.

**What we change:** Python API instead of YAML. Integrated with SQL workflows.

---

## Core Operators

```
┌─────────────────────────────────────────────────────────────┐
│  sem_map      Per-row semantic enrichment                   │
│  sem_filter   Semantic row filtering                        │
│  sem_match    Pairwise entity comparison                    │
│  sem_resolve  Full entity resolution pipeline               │
└─────────────────────────────────────────────────────────────┘
```

### Why these four?

- **sem_map**: The fundamental "enrich each row" operation. Classification, extraction, normalization.
- **sem_filter**: Semantic WHERE clause. Can't be done with SQL predicates.
- **sem_match**: Building block for joins and resolution. Explicit about candidate pairs.
- **sem_resolve**: The full pipeline most geo users actually need (POI deduplication, conflation).

### Why not more?

- `sem_agg`: Use SQL `GROUP BY` + `sem_map` on groups. Composition > operators.
- `sem_topk`: Use `sem_map` to score, then `ORDER BY + LIMIT`.
- `sem_join`: Implicit O(n²) is dangerous. `sem_match` on explicit candidates is safer.

**Minimal operator set. Maximum composability.**

---

## AI-Agent Friendliness

This library is designed to be used by AI agents, not just humans.

### What this means:

1. **Clear, typed interfaces** — Agents can understand function signatures
2. **Predictable behavior** — Same inputs → same outputs (with LLM mocking)
3. **Good error messages** — Agents can self-correct
4. **Comprehensive docstrings** — Context for agents working with the code
5. **Spec-driven tests** — `tests.yaml` is parseable by agents

### Future: Skills Package

Eventually, GeoSemOps will be packaged as an agent skill:

```
/geosemops resolve POIs from osm_pois and city_licenses
```

The skill provides context, examples, and guides the agent to use the library correctly.

---

## Repository as Example

This repo aims to demonstrate best practices for AI-assisted development:

```
geosemops/
├── INTENT.md          # This file. The "why" behind decisions.
├── SPEC.md            # Detailed behavior specification
├── tests.yaml         # Language-agnostic test cases
├── docs/
│   └── DESIGN.md      # Architecture and design decisions
├── src/geosemops/     # The actual library
└── tests/             # Pytest tests (generated from tests.yaml)
```

### Why this structure?

- **INTENT.md**: Captures reasoning. Survives beyond chat sessions.
- **SPEC.md**: Precise behavior spec. Agents can read and implement.
- **tests.yaml**: Test cases as data. Portable, parseable, checkable.
- **DESIGN.md**: Deeper technical details for contributors.

---

## Decisions Log

Significant decisions, captured as they're made.

### 2025-01-31: Spec-driven development

**Context**: Starting the project. Inspired by whenwords (library with no code, just specs).

**Decision**: Write specs and tests first, then implement. But we ARE writing code—not relying on per-install AI generation.

**Reasoning**:
- Specs clarify thinking before implementation
- Tests define behavior precisely
- But real code provides reliability, performance, maintenance

### 2025-01-31: DuckDB over Pandas

**Context**: LOTUS uses Pandas. Should we?

**Decision**: DuckDB-native.

**Reasoning**:
- SQL composes better with GIS tools (PostGIS, spatial SQL)
- DuckDB handles larger data without memory issues
- Agents write better SQL than Pandas
- Growing ecosystem (Motherduck, integrations)

### 2025-01-31: Four operators, not more

**Context**: LOTUS has 5 operators. DocETL has more. How many do we need?

**Decision**: Four core operators (map, filter, match, resolve).

**Reasoning**:
- Aggregation → SQL GROUP BY + sem_map
- TopK → sem_map for scoring + SQL ORDER BY LIMIT
- Join → explicit candidate pairs + sem_match (avoids hidden O(n²))
- Minimal API is easier to learn, use, and maintain

---

## Open Questions

Captured for future discussion.

- **Embedding model**: Local (sentence-transformers) vs API (OpenAI)? Tradeoff: cost/latency vs quality.
- **Caching strategy**: How aggressive? Per-prompt? Content-addressed?
- **Batch size**: What's optimal for different LLM providers?
- **Streaming**: Should operators support streaming for large datasets?

---

## Contributing

When making changes, update this document:

1. Add significant decisions to the **Decisions Log**
2. Move resolved questions from **Open Questions**
3. Update **Design Principles** if they evolve

The goal: anyone (human or AI) can understand not just *what* this code does, but *why*.
