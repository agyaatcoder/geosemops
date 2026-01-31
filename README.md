# GeoSemOps

**Semantic Operators for LLM-Powered Geospatial Data Processing**

GeoSemOps brings LLM-powered semantic operations to geospatial data workflows. Built on DuckDB with its spatial extension, it provides composable operators for tasks that were previously difficult with traditional GIS tools.

## Why GeoSemOps?

- **GIS tools** excel at geometry and topology (GDAL, PostGIS, DuckDB spatial)
- **LLMs** excel at understanding meaning, fuzzy matching, and classification
- **GeoSemOps** bridges the gap with declarative semantic operations

## Quick Start

```python
import geosemops as gso

# Connect to DuckDB with spatial extension
db = gso.connect("pois.duckdb")

# Normalize POI categories using an LLM
enriched = db.sem_map(
    relation="SELECT id, name, tags FROM raw_pois",
    columns=["name", "tags"],
    prompt="Classify this POI: {{ name }} with tags {{ tags }}",
    output_schema={"category": str, "is_chain": bool}
)

# Filter to actual cafes (not restaurants serving coffee)
cafes = db.sem_filter(
    relation=enriched,
    columns=["name", "tags", "category"],
    predicate="This is a true cafe/coffee shop, not a restaurant"
)
```

## Core Operators

| Operator | Purpose | Inspired By |
|----------|---------|-------------|
| `sem_map` | Per-row semantic enrichment | LOTUS sem_map |
| `sem_filter` | Semantic row filtering | LOTUS sem_filter |
| `sem_match` | Pairwise entity comparison | DocETL equijoin |
| `sem_resolve` | Full entity resolution pipeline | DocETL resolve |

## Installation

```bash
pip install geosemops
```

Requires Python 3.10+ and an LLM API key (OpenAI, Anthropic, etc. via LiteLLM).

```bash
export OPENAI_API_KEY="sk-..."
```

## Example: POI Conflation

Merge POIs from multiple sources (OSM, city data, Overture):

```python
import geosemops as gso

db = gso.connect()

# Load data from multiple sources
db.execute("""
    CREATE TABLE all_pois AS
    SELECT 'osm' as source, * FROM 'osm_pois.parquet'
    UNION ALL
    SELECT 'city' as source, * FROM 'city_licenses.parquet'
""")

# Resolve duplicates with spatial + semantic matching
entities = db.sem_resolve(
    relation="all_pois",
    blocking_keys=["name"],
    spatial_blocking={"geom": "geometry", "max_distance_m": 50},
    comparison_prompt="""
        Compare these POIs:
        1: {{ input1.name }} at {{ input1.address }}
        2: {{ input2.name }} at {{ input2.address }}
        Are they the same place?
    """,
    resolution_prompt="""
        Merge these duplicate records into one canonical POI.
        Prefer the most complete address and specific category.
    """
)
```

## Design Principles

1. **Minimal API** — Four core operators handle 90% of use cases
2. **Explicit blocking** — Candidate generation is visible and tunable
3. **SQL-native** — Composes naturally with DuckDB spatial queries
4. **Inspectable** — View prompts, intermediate results, and LLM decisions

## Roadmap

- [x] Design specification
- [ ] Phase 1: LLM UDFs + `sem_map`
- [ ] Phase 2: `sem_filter` + `sem_match`
- [ ] Phase 3: `sem_resolve` pipeline
- [ ] Phase 4: Evaluation + documentation

## Acknowledgments

Inspired by [LOTUS](https://github.com/stanford-futuredata/lotus) (Stanford/Berkeley) and [DocETL](https://github.com/ucbepic/docetl) (UC Berkeley).

See [docs/DESIGN.md](docs/DESIGN.md) for the full design specification.

## License

MIT
