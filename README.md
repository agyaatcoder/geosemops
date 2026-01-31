# GeoSemOps

**Semantic Operators for LLM-Powered Geospatial Data Processing**

GIS tools excel at geometry. LLMs excel at meaning. GeoSemOps bridges the gap.

## Quick Example

```python
import geosemops as gso

db = gso.connect("pois.duckdb")

# Normalize messy POI categories
enriched = db.sem_map(
    relation="SELECT * FROM raw_pois",
    columns=["name", "tags"],
    prompt="Classify this POI: {{ name }} ({{ tags }})",
    output_schema={"category": str, "is_chain": bool}
)

# Filter to actual cafes (not restaurants serving coffee)
cafes = db.sem_filter(
    relation=enriched,
    columns=["name", "category"],
    predicate="This is a true cafe, not a restaurant"
)

# Resolve duplicates across OSM + city data
entities, membership = db.sem_resolve(
    relation="SELECT * FROM all_pois",
    blocking_keys=["name"],
    spatial_blocking={"geom": "geometry", "max_distance_m": 50},
    comparison_prompt="Are these the same place? ...",
    resolution_prompt="Merge into canonical record: ..."
)
```

## Operators

| Operator | Purpose |
|----------|---------|
| `sem_map` | Enrich each row with LLM-generated columns |
| `sem_filter` | Filter rows using natural language conditions |
| `sem_match` | Compare pairs to find matching entities |
| `sem_resolve` | Full entity resolution: blocking → matching → clustering → canonicalization |

## Installation

```bash
pip install geosemops
```

Requires Python 3.10+ and an LLM API key:

```bash
export OPENAI_API_KEY="sk-..."
```

## Why GeoSemOps?

- **DuckDB-native** — SQL-composable, not a new paradigm
- **Spatial-first** — Geometry is a first-class citizen
- **Inspectable** — Prompts visible, blocking explicit
- **Minimal API** — Four operators handle 90% of use cases

See [INTENT.md](INTENT.md) for the full reasoning behind the design.

## Documentation

| Document | Purpose |
|----------|---------|
| [INTENT.md](INTENT.md) | Why this library exists, design decisions |
| [SPEC.md](SPEC.md) | Detailed behavior specification |
| [docs/DESIGN.md](docs/DESIGN.md) | Architecture and implementation details |
| [tests.yaml](tests.yaml) | Test cases as data |

## Inspiration

- [LOTUS](https://github.com/stanford-futuredata/lotus) — Semantic operators for DataFrames
- [DocETL](https://github.com/ucbepic/docetl) — LLM-powered document processing

## License

MIT
