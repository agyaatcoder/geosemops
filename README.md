# GeoSemOps

**Semantic Operators for LLM-Powered Geospatial Data Processing**

GeoSemOps is a library with no code—just specs and tests.

## What?

This repo contains:

- **[SPEC.md](SPEC.md)** — Detailed behavior specification
- **[tests.yaml](tests.yaml)** — Language-agnostic test cases
- **[INSTALL.md](INSTALL.md)** — Build instructions (a prompt)

You "install" GeoSemOps by asking an AI to implement it. The spec is tight enough that Claude, Codex, Cursor, or similar agents can build a working implementation in one shot.

## Why?

GIS tools excel at geometry. LLMs excel at meaning. GeoSemOps bridges the gap with four semantic operators:

| Operator | Purpose |
|----------|---------|
| `sem_map` | Enrich each row with LLM-generated columns |
| `sem_filter` | Filter rows using natural language conditions |
| `sem_match` | Compare pairs to find matching entities |
| `sem_resolve` | Full entity resolution pipeline |

## Quick Start

Paste into Claude Code, Cursor, or your AI coding agent:

```
Implement the geosemops library in Python.

1. Read SPEC.md for complete behavior specification
2. Parse tests.yaml and generate pytest test files
3. Implement all four operators: sem_map, sem_filter, sem_match, sem_resolve
4. Run tests until all pass
5. Place implementation in src/geosemops/
```

Then use it:

```python
import geosemops as gso

db = gso.connect("pois.duckdb")

# Normalize POI categories
enriched = db.sem_map(
    relation="SELECT * FROM raw_pois",
    columns=["name", "tags"],
    prompt="Classify this POI: {{ name }} ({{ tags }})",
    output_schema={"category": str, "is_chain": bool}
)

# Filter to actual cafes
cafes = db.sem_filter(
    relation=enriched,
    columns=["name", "category"],
    predicate="This is a true cafe, not a restaurant that happens to serve coffee"
)
```

## Why Spec-Driven?

Inspired by [whenwords](https://dbreunig.com/2025/01/08/software-library-with-no-code.html):

> "Do we need many language-specific implementations? Or do we need one tightly defined set of rules which we implement on demand?"

For simple utilities, the answer might be: just ship the spec.

GeoSemOps is more complex than whenwords (4 operators, DuckDB integration, embeddings), but the principle holds. The spec is ~500 lines. Tests are exhaustive. AI agents can implement it reliably.

### When this works

- ✅ Well-defined behavior (semantic operators have clear inputs/outputs)
- ✅ Standard dependencies (DuckDB, LiteLLM are cross-language)
- ✅ Testing is straightforward (mock LLM, check outputs)

### When this doesn't work

- ❌ Performance-critical code (you want battle-tested implementations)
- ❌ Complex state management (specs get unwieldy)
- ❌ Needs ongoing security updates (you want a maintained codebase)

## Design

See [docs/DESIGN.md](docs/DESIGN.md) for the full design specification including:

- Architecture and operator details
- Comparison with LOTUS and DocETL
- Implementation roadmap
- Open questions

## Inspiration

- [LOTUS](https://github.com/stanford-futuredata/lotus) — Semantic operators for DataFrames
- [DocETL](https://github.com/ucbepic/docetl) — LLM-powered document processing
- [whenwords](https://github.com/dbreunig/whenwords) — The spec-only library concept

## License

MIT
