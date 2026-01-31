# Installing GeoSemOps

GeoSemOps is a spec-driven library. You build it by prompting an AI coding agent.

## Quick Install

Copy this prompt into Claude, Cursor, Codex, or your preferred AI coding agent:

```
Implement the geosemops library in Python.

1. Read SPEC.md for complete behavior specification
2. Parse tests.yaml and generate pytest test files
3. Implement all four operators: sem_map, sem_filter, sem_match, sem_resolve
4. Implement the connect() function and GeoSemConnection class
5. Run tests until all pass
6. Place implementation in src/geosemops/

Use these dependencies:
- duckdb (with spatial extension)
- litellm (for LLM calls)
- jinja2 (for prompt templating)
- sentence-transformers (for embeddings in sem_resolve)
- pydantic (for schema validation)

All tests.yaml test cases must pass. Mock LLM responses as specified in each test.
```

## Manual Install (if you prefer)

```bash
pip install geosemops
```

## Requirements

- Python 3.10+
- LLM API key (OpenAI, Anthropic, etc.)

```bash
export OPENAI_API_KEY="sk-..."
# or
export ANTHROPIC_API_KEY="sk-ant-..."
```

## Verify Installation

```python
import geosemops as gso

db = gso.connect()
result = db.sem_map(
    relation="SELECT 1 as id, 'Test Cafe' as name",
    columns=["name"],
    prompt="Classify: {{ name }}",
    output_schema={"category": str}
)
print(result.fetchall())
# [(1, 'Test Cafe', 'cafe')]
```

## Alternative Languages

While the reference implementation is Python, the spec is language-agnostic. To implement in another language:

```
Implement the geosemops library in [LANGUAGE].

1. Read SPEC.md for complete behavior specification
2. Parse tests.yaml and generate tests in [LANGUAGE]'s test framework
3. Implement all four operators: sem_map, sem_filter, sem_match, sem_resolve
4. Integrate with DuckDB (available for most languages)
5. Use an LLM client library for [LANGUAGE]
6. Run tests until all pass

All tests.yaml test cases must pass.
```

Tested with: Python, TypeScript (Deno), Rust, Go

## Customization

After generation, you may want to:

1. **Adjust batching**: Modify `GEOSEMOPS_BATCH_SIZE` for your use case
2. **Change models**: Set `GEOSEMOPS_MODEL` to your preferred LLM
3. **Add caching**: The spec defines caching behavior, but you can extend it
4. **Optimize blocking**: Tune `blocking_threshold` for your data

## Troubleshooting

**Tests fail on LLM mocking**: Ensure your test harness matches `input_contains` before returning `response`

**DuckDB spatial not found**: Run `INSTALL spatial; LOAD spatial;` in DuckDB

**Rate limits**: Increase batch delays or use a different model
