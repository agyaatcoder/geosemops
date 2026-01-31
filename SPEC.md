# GeoSemOps Specification

**Semantic Operators for LLM-Powered Geospatial Data Processing**
*Version 1.0*

---

## Overview

GeoSemOps provides four semantic operators that use LLMs to process geospatial data. All operators work with DuckDB and its spatial extension.

## Core Concepts

### LLM Integration
- All operators use LiteLLM for multi-provider LLM support
- Prompts use Jinja2 templating with column placeholders: `{{ column_name }}`
- Responses are parsed as structured JSON matching the specified schema

### Geometry Handling
- Geometry columns use DuckDB's GEOMETRY type
- When passed to LLM, geometries are converted to WKT via `ST_AsText()`
- Spatial operations use DuckDB spatial extension functions

---

## Operators

### 1. sem_map

**Purpose:** Apply LLM to each row to produce additional columns.

**Signature:**
```
sem_map(
    relation: str | Relation,
    columns: list[str],
    prompt: str,
    output_schema: dict[str, type]
) -> Relation
```

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| relation | str or Relation | SQL query or table name |
| columns | list[str] | Columns to include in LLM context |
| prompt | str | Jinja2 template with `{{ column }}` placeholders |
| output_schema | dict | Maps output column names to Python types (str, int, float, bool) |

**Behavior:**
1. Execute `relation` to get input rows
2. For each row:
   a. Build context dict from specified `columns`
   b. Render `prompt` with Jinja2 using context
   c. Call LLM with rendered prompt, request JSON matching `output_schema`
   d. Parse LLM response into output columns
3. Return relation with original columns + new output columns

**Example:**
```python
sem_map(
    relation="SELECT id, name, tags FROM pois",
    columns=["name", "tags"],
    prompt="Classify this POI into a category.\nName: {{ name }}\nTags: {{ tags }}",
    output_schema={"category": str, "confidence": float}
)
```

**Output:** Original columns plus `category` (str) and `confidence` (float)

---

### 2. sem_filter

**Purpose:** Keep rows where LLM returns TRUE for a natural language predicate.

**Signature:**
```
sem_filter(
    relation: str | Relation,
    columns: list[str],
    predicate: str
) -> Relation
```

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| relation | str or Relation | SQL query or table name |
| columns | list[str] | Columns to include in LLM context |
| predicate | str | Natural language condition (Jinja2 template) |

**Behavior:**
1. Execute `relation` to get input rows
2. For each row:
   a. Build context dict from specified `columns`
   b. Render `predicate` with Jinja2 using context
   c. Call LLM asking: "Does this satisfy the condition? Answer TRUE or FALSE."
   d. Parse response as boolean
3. Return only rows where LLM returned TRUE

**Example:**
```python
sem_filter(
    relation="SELECT * FROM parks",
    columns=["name", "description", "amenities"],
    predicate="This park is suitable for large outdoor concerts"
)
```

---

### 3. sem_match

**Purpose:** Given candidate pairs, determine which pairs represent the same entity.

**Signature:**
```
sem_match(
    candidate_pairs: str | Relation,
    left_columns: list[str],
    right_columns: list[str],
    comparison_prompt: str
) -> Relation
```

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| candidate_pairs | str or Relation | Relation with columns for both records |
| left_columns | list[str] | Column names for left record (use `left_` prefix in data) |
| right_columns | list[str] | Column names for right record (use `right_` prefix in data) |
| comparison_prompt | str | Jinja2 template using `{{ left.col }}` and `{{ right.col }}` |

**Behavior:**
1. Execute `candidate_pairs` to get pair rows
2. For each pair:
   a. Build `left` context from columns matching `left_columns`
   b. Build `right` context from columns matching `right_columns`
   c. Render `comparison_prompt` with both contexts
   d. Call LLM asking if records match, request JSON `{is_match: bool, confidence: float}`
3. Return pairs with added columns: `is_match` (bool), `match_confidence` (float)

**Important:** This operator does NOT perform blocking. Generate candidate pairs using SQL:
```sql
SELECT a.*, b.*
FROM table_a a, table_b b
WHERE ST_DWithin(a.geom, b.geom, 100)  -- spatial blocking
```

**Example:**
```python
sem_match(
    candidate_pairs="SELECT * FROM poi_candidates",
    left_columns=["left_name", "left_address"],
    right_columns=["right_name", "right_address", "distance_m"],
    comparison_prompt="""
        Compare these business records:
        Business 1: {{ left.left_name }} at {{ left.left_address }}
        Business 2: {{ right.right_name }} at {{ right.right_address }}
        Distance: {{ right.distance_m }}m

        Are these the same business?
    """
)
```

---

### 4. sem_resolve

**Purpose:** Full entity resolution: blocking → matching → clustering → canonicalization.

**Signature:**
```
sem_resolve(
    relation: str | Relation,
    blocking_keys: list[str],
    comparison_prompt: str,
    resolution_prompt: str,
    blocking_threshold: float = 0.7,
    spatial_blocking: dict | None = None,
    output_schema: dict | None = None
) -> tuple[Relation, Relation]
```

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| relation | str | Input table with potential duplicates |
| blocking_keys | list[str] | Columns for embedding similarity blocking |
| comparison_prompt | str | Prompt for pairwise comparison (`{{ input1.* }}`, `{{ input2.* }}`) |
| resolution_prompt | str | Prompt for canonicalizing clusters (`{{ inputs }}` is list) |
| blocking_threshold | float | Min embedding similarity 0.0-1.0 (default 0.7) |
| spatial_blocking | dict | Optional `{"geom": "col_name", "max_distance_m": 50}` |
| output_schema | dict | Schema for canonical output (default: infer from input) |

**Behavior:**

**Phase 1: Blocking**
1. Generate embeddings for `blocking_keys` columns (concatenated as text)
2. Find pairs with embedding similarity >= `blocking_threshold`
3. If `spatial_blocking` provided, also require `ST_DWithin(geom, geom, max_distance_m)`
4. Candidate pairs = intersection of embedding and spatial blocks

**Phase 2: Matching**
1. For each candidate pair, render `comparison_prompt` with `input1` and `input2`
2. Call LLM for match decision (same as sem_match)
3. Build match graph: nodes = records, edges = matched pairs

**Phase 3: Clustering**
1. Run connected components (union-find) on match graph
2. Each component = one resolved entity

**Phase 4: Canonicalization**
1. For each cluster, render `resolution_prompt` with `inputs` list
2. Call LLM to produce canonical record matching `output_schema`

**Returns:** Tuple of two relations:
- `entities`: Canonical records with `entity_id` and schema columns
- `membership`: Maps `original_id` → `entity_id`

**Example:**
```python
entities, membership = sem_resolve(
    relation="SELECT * FROM all_pois",
    blocking_keys=["name", "address"],
    blocking_threshold=0.65,
    spatial_blocking={"geom": "geometry", "max_distance_m": 50},
    comparison_prompt="""
        Are these the same place?
        POI 1: {{ input1.name }} | {{ input1.address }}
        POI 2: {{ input2.name }} | {{ input2.address }}
    """,
    resolution_prompt="""
        Merge these duplicate records into one canonical POI:
        {% for poi in inputs %}
        - {{ poi.name }} | {{ poi.address }} | Source: {{ poi.source }}
        {% endfor %}

        Return the best name, most complete address, and primary category.
    """,
    output_schema={"name": str, "address": str, "category": str}
)
```

---

## Connection API

### connect()

**Signature:**
```
connect(database: str | None = None) -> GeoSemConnection
```

**Behavior:**
1. Create DuckDB connection (in-memory if `database` is None)
2. Install and load spatial extension
3. Return connection wrapper with semantic operator methods

**Example:**
```python
import geosemops as gso

db = gso.connect("pois.duckdb")
db.sem_map(...)
db.sem_filter(...)
```

---

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `GEOSEMOPS_MODEL` | LLM model identifier | `gpt-4o-mini` |
| `GEOSEMOPS_EMBEDDING_MODEL` | Embedding model | `text-embedding-3-small` |
| `GEOSEMOPS_CACHE_DIR` | Response cache directory | `.geosemops_cache` |
| `GEOSEMOPS_BATCH_SIZE` | Rows per LLM batch | `10` |

### LLM Provider

Uses LiteLLM, so any provider works via environment:
- OpenAI: `OPENAI_API_KEY`
- Anthropic: `ANTHROPIC_API_KEY`
- Azure: `AZURE_API_KEY`, `AZURE_API_BASE`

---

## Error Handling

### LLM Failures
- Retry up to 3 times with exponential backoff
- On persistent failure, return `None` for output columns (sem_map) or `False` (sem_filter/sem_match)

### Schema Validation
- If LLM response doesn't match `output_schema`, retry with explicit schema reminder
- After 2 retries, return `None` values

### Geometry Errors
- Invalid geometries are passed as `NULL` to LLM context
- ST_* functions follow DuckDB spatial error handling

---

## Testing

Test cases are defined in `tests.yaml` as input/output pairs.

### Test Format
```yaml
- name: test_name
  operator: sem_map | sem_filter | sem_match | sem_resolve
  input:
    relation: "SQL or table data"
    columns: [...]
    prompt: "..."
    # operator-specific params
  mock_llm_responses:
    - {input_contains: "text", response: {...}}
  expected_output:
    columns: [...]
    rows: [[...], [...]]
```

### Running Tests
Tests mock LLM calls using `mock_llm_responses`. Implementations must:
1. Parse `tests.yaml`
2. Set up mock LLM that matches `input_contains` and returns `response`
3. Run operator with `input` params
4. Assert output matches `expected_output`

---

## Appendix: Type Mappings

| Python Type | DuckDB Type | JSON Type |
|-------------|-------------|-----------|
| str | VARCHAR | string |
| int | INTEGER | number |
| float | DOUBLE | number |
| bool | BOOLEAN | boolean |
| list[str] | VARCHAR[] | array |
