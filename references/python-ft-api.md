# Python FT (Search) Module API Reference

Use when writing or reviewing Python code that calls ft.create, ft.search, ft.aggregate, ft.hybrid, ft.cursor_read/cursor_del, ft.dropindex, or ft.info.

## Contents

- [Key Rules](#key-rules) · [Server Version Matrix](#server-version-matrix-valkey-search-12-vs-13) · [Import Pattern](#import-pattern)
- [Field Constructors](#field-constructor-signatures) · [ft.create()](#ftcreate) · [ft.search()](#ftsearch) · [Vector Range](#vector-range-queries-search--13)
- [ft.aggregate()](#ftaggregate) · [Reducers](#reducers) · [Cursors](#cursors-ftcursor_read--ftcursor_del-search--13) · [ft.hybrid()](#fthybrid-search--13)
- [ft.info()](#ftinfo) · [Commands the Server Does Not Implement](#commands-the-server-does-not-implement) · [Common Mistakes](#common-mistakes)

**Citation legend.** `G:` = valkey-glide @ `b7c92e92b`, file under `python/glide-shared/glide_shared/commands/server_modules/ft_options/`. `GA:` = `python/glide-async/python/glide/async_commands/ft.py` @ `b7c92e92b` (the sync twin `glide_sync/sync_commands/ft.py` has identical signatures). `GT:` = `python/tests/async_tests/tests_server_modules/test_ft.py` @ `b7c92e92b`. `VS:` = valkey-search @ `1.3.0-rc1` (`e31273a`).

## Key Rules

1. **NO CLIENT METHODS EXIST**: `client.ft_search()`, `client.ft_create()`, `client.ft()` DO NOT EXIST
2. **ALL FUNCTIONS ARE MODULE-LEVEL**: Import `ft` and call `ft.function(client, ...)`
3. **DO NOT INFER FROM REDIS-PY**: This is NOT Redis-py. Redis-py patterns DO NOT apply.
4. **THIS IS THE ONLY DOCUMENTATION**: Vector search API is not documented elsewhere. Do not infer usage from other sources.
5. **KNOW THE SERVER VERSION**: several rules below differ between valkey-search 1.2 and 1.3, and some 1.3 behaviour also depends on `search.emulate-release`. Check both before choosing a pattern.

## Server Version Matrix (valkey-search 1.2 vs 1.3)

Two things decide behaviour: the **module version** (1.2 or 1.3) and, on 1.3, the **`search.emulate-release`** config. Compatibility fixes that change a reply or a parse are gated on `search.emulate-release >= 1.3.0`; the default is the current major `.0.0`, i.e. **`1.0.0` for all of 1.x**, so a stock 1.3 server keeps the 1.2 behaviour for every gated item ‹VS:src/valkey_search_options.cc:876,896-901; VS:COMPATIBILITY.md:172›. Opt in with `CONFIG SET search.emulate-release 1.3.0` (accepted because it is ≤ the module version ‹VS:src/valkey_search_options.cc:887-894; VS:src/version.h:23›; the same CONFIG SET mechanism is used e.g. in ‹VS:integration/compatibility_test.py:984›).

| Topic | search < 1.3 | search ≥ 1.3, default emulate-release (1.0.0) | search ≥ 1.3, emulate-release ≥ 1.3.0 |
|---|---|---|---|
| Bare `*` query (FT.SEARCH / FT.AGGREGATE) | `RequestError: Invalid query string syntax` — use `@num:[-inf +inf]` or a tag filter | Match-all (#960, not gated) ‹VS:src/query/search.cc:2447-2450; GT:2397,2500› | same |
| Reducer / SORTBY / APPLY / FILTER fields without LOAD | Silently read as missing (SUM → 0) | Auto-loaded (#1341, not gated; vector fields excluded) ‹VS:src/commands/ft_aggregate.cc:50-93; GT:2500› | same |
| `LoadField(..., alias=...)` (`LOAD … AS`) | Error | Error: ``Index: field `AS` does not exist`` ‹GT:2280› | Column renamed ‹VS:COMPATIBILITY.md:192; GT:2280› |
| SORTBY retained rows with adjacent LIMIT | SORTBY keeps its MAX (default 10) | same as < 1.3 | Adjacent LIMIT widens to offset+count ‹VS:COMPATIBILITY.md:201; G:ft_aggregate_options.py:30-42; GT:2912› |
| REDUCE without `name` → column name | `COUNT_DISTINCT(@rating)` style | same | `__generated_aliascount_distinctrating` style ‹VS:COMPATIBILITY.md:189› |
| FT.INFO `index_definition` | 6 elements, `default_score` = `"1"`, no `score_field` | same | 8 elements, real `default_score`, `score_field` pair ‹VS:COMPATIBILITY.md:190; GA:326-329; GT:3299› |
| JSON index FT.AGGREGATE column names | Schema identifier (`$.price`) | same | As written in the query (`@price` → `price`) ‹VS:COMPATIBILITY.md:191› |
| `in_keys`, VECTOR_RANGE, cursors, FT.HYBRID, TOLIST/FIRST_VALUE/RANDOM_SAMPLE, FLOAT16/BFLOAT16, FILTER, SCORE_FIELD, WITHSCORES/SCORER, ADDSCORES, SORTABLE UNF, NOHL | Not available | Available ‹VS:00-RELEASENOTES "New Features"› | same |

Other gated 1.3.0 fixes (division by zero, `^`/`&&` precedence, MIN/MAX numeric, SUM/AVG of empty group → `nan`, apostrophe phrases, stemming/fuzzy by code point) are listed in ‹VS:COMPATIBILITY.md:188-204›. Code that must run on both levels should not depend on either branch of a gated item.

## Import Pattern

```python
# Sync — top-level imports (preferred in v2.3+)
from glide_sync import (
    ft, GlideClient,
    FtCreateOptions, DataType, DistanceMetricType,
    VectorField, VectorFieldAttributesFlat, VectorFieldAttributesHnsw,
    VectorAlgorithm, VectorType,
    TagField, NumericField, TextField,
    FtSearchOptions, FtSearchLimit, ReturnField,
)

# Async — same names from glide
from glide import (
    ft, GlideClient,
    FtCreateOptions, DataType, DistanceMetricType,
    VectorField, VectorFieldAttributesFlat, VectorFieldAttributesHnsw,
    VectorAlgorithm, VectorType,
    TagField, NumericField, TextField,
    FtSearchOptions, FtSearchLimit, ReturnField,
)

# Deep imports also work (legacy, still valid):
from glide_shared.commands.server_modules.ft_options.ft_create_options import (
    FtCreateOptions, TextField, VectorField, VectorFieldAttributesFlat,
    VectorFieldAttributesHnsw, VectorAlgorithm, DistanceMetricType, VectorType, DataType,
)
from glide_shared.commands.server_modules.ft_options.ft_search_options import (
    FtSearchOptions, FtSearchLimit, ReturnField,
)

# WRONG - These do not exist:
# from glide_sync import FT  # NO — lowercase 'ft' only
# client.ft  # NO
# client.ft_search  # NO
# ft.FtCreateOptions  # NO — not public API; import FtCreateOptions directly
```

**valkey-search 1.3 names** (exported from both `glide` and `glide_sync` `__all__` ‹python/glide-async/python/glide/__init__.py:389,458-486; python/glide-sync/glide_sync/__init__.py:327,401-429›):

```python
from glide import (  # or glide_sync
    LoadField, FtWithCursor,                                  # aggregate LOAD … AS, WITHCURSOR
    FtHybridSearchArm, FtHybridVsimArm, FtHybridVsimFilter,   # ft.hybrid arms
    FtHybridOptions, FtHybridCombine, FtHybridRrf, FtHybridLinear, FtHybridFunction,
    HybridFilterPolicy,
    InfoScope, ShardScope, ConsistencyMode, OrderBy,
)
```

## Distance Metrics Mapping

```python
from glide_sync import DistanceMetricType  # top-level import (v2.3+)

distance_map = {
    "COSINE": DistanceMetricType.COSINE,
    "L2": DistanceMetricType.L2,
    "IP": DistanceMetricType.IP,
}
```

## Field Constructor Signatures

### TextField
```python
TextField(
    name: TEncodable,
    alias: Optional[TEncodable] = None,
    nostem: bool = False,
    weight: Optional[float] = None,
    withsuffixtrie: bool = False,
    nosuffixtrie: bool = False,
    sortable: bool = False,
    unf: bool = False,         # search ≥ 1.3; requires sortable=True (ValueError otherwise)
)
```

### TagField
```python
TagField(
    name: TEncodable,
    alias: Optional[TEncodable] = None,
    separator: Optional[TEncodable] = None,
    case_sensitive: bool = False,
    sortable: bool = False,
    unf: bool = False,         # search ≥ 1.3; requires sortable=True
)
```

### NumericField
```python
NumericField(
    name: TEncodable,
    alias: Optional[TEncodable] = None,
    sortable: bool = False,
    unf: bool = False,         # search ≥ 1.3; requires sortable=True
)
```

Signatures ‹G:ft_create_options.py:171,232,278›; `unf` without `sortable` raises `ValueError` ‹G:ft_create_options.py:201,257,298›.

**`unf=True` is a no-op on valkey-search 1.3.** It is accepted, stored and reported by FT.INFO (`UNF` = `1`), but TEXT/TAG sort keys are still compared byte-for-byte (`"Banana"` sorts before `"apple"`) ‹GT:3235›. Include it only for RediSearch call-shape compatibility; never rely on it to change sort order.

### The `sortable` Parameter

All three field types accept `sortable: bool = False`. When set to `True`, the field value can be used for SORTBY in FT.SEARCH results. This enables sorting search results by that field's value.

```python
# Example: create index with sortable fields
schema = [
    TextField("title", sortable=True),
    NumericField("price", sortable=True),
    TagField("category", sortable=True),
]
```

## Core Functions

### ft.create()

**Creates a search index.**

```python
ft.create(
    client: GlideClient,
    index_name: str,
    schema: List[Field],
    options: Optional[FtCreateOptions] = None
) -> str  # Returns "OK"
```

**Parameters:**
- `client`: GlideClient instance (first parameter, always required)
- `index_name`: Name for the index
- `schema`: List of Field objects (TextField, VectorField, etc.)
- `options`: FtCreateOptions with prefixes, data type, etc.

**Complete Example:**
```python
from glide_sync import (
    ft, GlideClient,
    FtCreateOptions, TextField, VectorField,
    VectorFieldAttributesFlat, VectorFieldAttributesHnsw,
    VectorAlgorithm, DistanceMetricType, VectorType, DataType,
)

# Define schema with FLAT algorithm
schema_flat = [
    TextField("title"),
    VectorField(
        "embedding",
        VectorAlgorithm.FLAT,
        VectorFieldAttributesFlat(
            dimensions=768,
            distance_metric=DistanceMetricType.COSINE,
            type=VectorType.FLOAT32
        )
    )
]

# Define schema with HNSW algorithm
schema_hnsw = [
    TextField("title"),
    VectorField(
        "embedding",
        VectorAlgorithm.HNSW,
        VectorFieldAttributesHnsw(
            dimensions=768,
            distance_metric=DistanceMetricType.COSINE,
            type=VectorType.FLOAT32,
            number_of_edges=16,
            vectors_examined_on_construction=200,
            vectors_examined_on_runtime=10,
        )
    )
]

# Create index - NOTE: ft.create(client, ...) NOT client.ft_create(...)
result = ft.create(
    client=client,
    index_name="products_idx",
    schema=schema_hnsw,
    options=FtCreateOptions(data_type=DataType.HASH, prefixes=["product:"])
)
# Returns: "OK"
```

**Common Mistakes:**
```python
# Wrong - Method does not exist
await client.ft_create("idx", schema)

# Wrong - No ft attribute on client
client.ft.create("idx", schema)

# Correct - Module-level function
ft.create(client, "idx", schema)
```

**FtCreateOptions — full signature** ‹G:ft_create_options.py:502›:
```python
FtCreateOptions(
    data_type=None, prefixes=None, score=None, language=None, skipinitialscan=False,
    minstemsize=None, withoffsets=False, nooffsets=False, nostopwords=False, stopwords=None,
    punctuation=None,
    filter=None,          # search ≥ 1.3: ingestion-time boolean expression
    score_field=None,     # search ≥ 1.3: per-document score field (boosts ranking)
    no_highlight=False,   # search ≥ 1.3: NOHL — accepted, a no-op (not even stored)
)
```
Mutually exclusive pairs raise `ValueError`: `withoffsets`/`nooffsets`, `nostopwords`/`stopwords`; `filter=""` also raises ‹G:ft_create_options.py:579-583›.

**search ≥ 1.3 — FILTER: index only matching keys** ‹GT:3194›
```python
ft.create(client, "people_idx", [NumericField("age")],
          FtCreateOptions(data_type=DataType.HASH, prefixes=["person:"], filter="@age > 18"))
# person:minor {age: 10} matches the prefix but is NOT indexed
```

**search ≥ 1.3 — SCORE_FIELD: per-document boost** ‹GT:3546›
```python
ft.create(client, "docs_idx", [TextField("title")],
          FtCreateOptions(data_type=DataType.HASH, prefixes=["doc:"], score_field="boost"))
# doc with boost=100.0 ranks above an otherwise identical doc with boost=1.0
```

**search ≥ 1.3 — FLOAT16 / BFLOAT16 vectors** (2 bytes per element ‹G:ft_create_options.py:81,91›). GLIDE never encodes vectors for you, for any type — pack the bytes yourself ‹GT:3123›:
```python
import struct

VectorFieldAttributesFlat(dimensions=2, distance_metric=DistanceMetricType.L2,
                          type=VectorType.FLOAT16)
fp16  = b"".join(struct.pack("<e", v) for v in values)          # FLOAT16
bf16  = b"".join(struct.pack("<f", v)[2:4] for v in values)     # BFLOAT16 (truncating; for HASH the server stores these bytes as-is)
# Store and query with the SAME encoding as the declared type
```
The server converts and rounds (ties-to-even) only when it parses floats itself, i.e. JSON arrays ‹VS:src/indexes/vector_type.cc:172-175; VS:src/indexes/bfloat16.h:20›. **BFLOAT16 needs a SIMD BF16 path on the server CPU**; elsewhere FT.CREATE fails with a FailedPrecondition error ‹VS:src/index_schema.cc:175-191; VS:src/indexes/vector_base.cc:660-684›. Prefer FLOAT16 when the fleet's CPUs are unknown.

### ft.search()

**Searches an index using a query.**

```python
ft.search(
    client: GlideClient,
    index_name: str,
    query: str,
    options: Optional[FtSearchOptions] = None
) -> FtSearchResponse  # [count, {key: {field: value}}]
```

**Parameters:**
- `client`: GlideClient instance (first parameter, always required)
- `index_name`: Name of the index to search
- `query`: Search query string (e.g., "hello", "@price:[0 100]", "*" on search ≥ 1.3, "*=>[KNN 5 @vec $query]")
- `options`: FtSearchOptions with params, return fields, etc.

**Return Format:**
```python
# Returns: [count, {key: {field: value}}]
# Example: [2, {b'doc:1': {b'title': b'Hello'}, b'doc:2': {b'title': b'World'}}]
```

**Basic Search Example:**
```python
from glide_sync import ft, GlideClient

# Simple search - NOTE: ft.search(client, ...) NOT client.ft_search(...)
# query="*" needs search ≥ 1.3; on 1.2 use "@price:[-inf +inf]" (see "Match-all query" below)
results = ft.search(
    client=client,
    index_name="products_idx",
    query="*",
    options=None
)

count = results[0]  # Number of results
docs = results[1]   # Dict of documents
```

**Vector Search Example:**
```python
from glide_sync import ft, GlideClient
from glide_shared.commands.server_modules.ft_options.ft_search_options import (
    FtSearchOptions
)
import struct

# Convert embedding to bytes
query_embedding = [0.1, 0.2, 0.3, ...]  # Your embedding
embedding_bytes = b''.join(struct.pack('f', x) for x in query_embedding)

# Vector search - NOTE: ft.search(client, ...) NOT client.ft_search(...)
results = ft.search(
    client=client,
    index_name="products_idx",
    query="*=>[KNN 5 @embedding $vec]",
    options=FtSearchOptions(params={"vec": embedding_bytes})
)

# Decode results (they are bytes)
for key, fields in results[1].items():
    str_key = key.decode() if isinstance(key, bytes) else key
    # See the section on 'Binary Data Handling' for complete decoding
```

**FT.SEARCH: Wildcard `*` cannot follow a filter expression**

When combining a filter with a text query, `*` after a filter is rejected as invalid syntax.

```python
# Wrong — raises RequestError: Invalid wildcard '*' markers
query = "(@category:{Electronics}) *"

# Correct — filter alone acts as match-all within the filter
query = "@category:{Electronics}"

# Correct — combine filter with actual text query
query = "(@category:{Electronics}) headphones"
```

**Helper pattern:**
```python
def build_text_query(query_text: str, filter_expr: str | None) -> str:
    if filter_expr and query_text.strip() == '*':
        return filter_expr
    if filter_expr:
        return f'({filter_expr}) {query_text}'
    return query_text
```

**SECURITY: Sanitize user input before query construction**

The `=>` token in FT.SEARCH syntax separates a filter expression from a KNN vector clause. If user-controlled input (e.g., a filter parameter from an API or MCP tool) contains `=>`, an attacker can inject a KNN vector search that bypasses all intended filters and returns all documents. This is a confirmed query injection vulnerability.

**Always reject `=>` in user-supplied filter expressions and vector field names:**
```python
def sanitize_search_input(filter_expr: str | None, vector_field: str | None = None) -> dict | None:
    """Returns error dict if input contains injection payload, None if safe."""
    if filter_expr and '=>' in filter_expr:
        return {'status': 'error', 'reason': "filter must not contain '=>'"}
    if vector_field and '=>' in vector_field:
        return {'status': 'error', 'reason': "vector_field must not contain '=>'"}
    return None
```

Call this before constructing any FT.SEARCH query from user input:
```python
err = sanitize_search_input(filter_expr, vector_field)
if err:
    return err
query = build_text_query(query_text, filter_expr)
```

**Common Mistakes:**
```python
# Wrong - Method does not exist
results = await client.ft_search("idx", "*")

# Wrong - No ft attribute on client  
results = client.ft.search("idx", "*")

# Correct - Module-level function
results = ft.search(client, "idx", "*")
```

**Pagination with FtSearchLimit:**

`FtSearchOptions` does not accept `offset` or `limit` as direct keyword arguments. Pagination requires wrapping in `FtSearchLimit`.

```python
# Wrong — TypeError: unexpected keyword argument
FtSearchOptions(first_result=0, limit=10)
```

```python
# Correct — use FtSearchLimit
from glide_shared.commands.server_modules.ft_options.ft_search_options import (
    FtSearchLimit,
    FtSearchOptions,
)

options = FtSearchOptions(limit=FtSearchLimit(offset=0, count=10))
results = ft.search(client=client, index_name="idx", query="*", options=options)  # "*" needs search ≥ 1.3
```

An offset past the end returns a bare `[total_count]` (one element, no dict) — guard `len(results) > 1` before reading `results[1]` ‹GT:3053›. `FtSearchOptions(count=True)` sends `LIMIT 0 0` (count only) and is mutually exclusive with `limit` (`ValueError`) ‹G:ft_search_options.py:150-158›.

**Match-all query is version-dependent** (applies to FT.SEARCH and FT.AGGREGATE alike — they share the query parser)

```python
# search ≥ 1.3 — bare "*" is match-all (#960) ‹GT:2397›
ft.search(client, "idx", "*")

# search < 1.3 — bare non-vector "*" raises RequestError: Invalid query string syntax
# ‹VS:src/query/search.cc:2447-2450 — the is_match_all exemption is what #960 (f1db9ed) added›.
# Use a range on an indexed NUMERIC field (works on every version):
ft.search(client, "idx", "@price:[-inf +inf]")

# KNN queries "*=>[KNN k @vec $blob]" were never affected — the "*" there is a pre-filter, not a bare query.
```

**search ≥ 1.3 — `in_keys` (INKEYS): restrict to given keys** ‹G:ft_search_options.py:173-178,254-258; GT:2432›

`None` (default), `[]` and a list are three different requests:

| `in_keys=` | Wire | Result |
|---|---|---|
| `None` | no INKEYS | no restriction |
| `[]` | `INKEYS 0` | **zero results**, always |
| `["k1", b"k2"]` | `INKEYS 2 k1 k2` | only those keys (post-filter; count reflects the filtered set) |

```python
# Wrong — "if caller passed nothing, send []" turns "no filter" into "match nothing"
FtSearchOptions(in_keys=allowed_keys or [])

# Correct — pass None to mean "no restriction"
FtSearchOptions(in_keys=allowed_keys if allowed_keys else None)
```

**search ≥ 1.3 — relevance scores** ‹GT:3379,3441›

`with_scores=True` changes each document value from a field dict to `[score, field_dict]`; with `withsortkeys=True` as well it is `[score, sort_key, field_dict]`. `scorer="BM25STD"` is the only scorer; `"TFIDF"` is rejected ‹GT:4604›.

```python
res = ft.search(client, "idx", "hello", FtSearchOptions(with_scores=True, scorer="BM25STD"))
for key, (score, fields) in res[1].items():
    print(key, float(score), fields)
```

### Vector Range Queries (search ≥ 1.3)

`@field:[VECTOR_RANGE <radius> $blob]` returns every document within `radius`, in both `ft.search` and `ft.aggregate`. Combine with AND (space), OR (`|`) and negation (`-`); name the distance with `=>{$yield_distance_as: <name>}` ‹GT:2645,2774›.

```python
q = array.array("f", [0.0, 0.0]).tobytes()

ft.search(client, "idx", "@vec:[VECTOR_RANGE 2 $blob]",
          FtSearchOptions(params={"blob": q}, nocontent=True))

ft.search(client, "idx", "@vec:[VECTOR_RANGE $r $blob]=>{$yield_distance_as: dist}",
          FtSearchOptions(params={"blob": q, "r": "0.5"}, return_fields=[ReturnField("dist")]))

ft.search(client, "idx", "@vec:[VECTOR_RANGE 2 $blob] @category:{a}",
          FtSearchOptions(params={"blob": q}, nocontent=True))

ft.aggregate(client, "idx", "@vec:[VECTOR_RANGE 3 $blob]=>{$yield_distance_as: dist}",
             FtAggregateOptions(loadFields=["dist"], params={"blob": q},
                                clauses=[FtAggregateSortBy([FtAggregateSortProperty("@dist", OrderBy.ASC)])]))
```

Server limits on 1.3 ‹VS:src/query/search.cc:2492-2504›:
- **No VECTOR_RANGE inside a KNN pre-filter**: `"@vec:[VECTOR_RANGE 5 $b]=>[KNN 1 @vec $b]"` → `RequestError` (`…not supported in the filter of a KNN query`) ‹GT:2645›. Run the range query, or the KNN, not one inside the other.
- **At most one VECTOR_RANGE per query.**
- For a range query, the distance appears in results only when named via `$yield_distance_as` (and listed in RETURN, if RETURN is used) ‹VS:docs/commands/ft.search.md:64,72›.

### ft.dropindex()

**Drops an index.**

```python
ft.dropindex(client: GlideClient, index_name: str) -> str  # Returns "OK"
```

**Example:**
```python
from glide_sync import ft

# NOTE: ft.dropindex(client, ...) NOT client.ft_dropindex(...)
result = ft.dropindex(client, "products_idx")
```

### ft.list()

**Lists all indexes.**

```python
ft.list(client: GlideClient) -> List[bytes]  # Returns list of index names
```

**Example:**
```python
from glide_sync import ft

# NOTE: ft.list(client) NOT client.ft_list()
indexes = ft.list(client)
# Returns: [b'idx1', b'idx2']
```

### ft.info()

**Gets information about an index.**

```python
ft.info(
    client: GlideClient,
    index_name: str,
    scope: Optional[InfoScope] = None,              # LOCAL | PRIMARY | CLUSTER
    shard_scope: Optional[ShardScope] = None,       # ALLSHARDS | SOMESHARDS
    consistency: Optional[ConsistencyMode] = None,  # CONSISTENT | INCONSISTENT
) -> FtInfoResponse
```
‹GA:305; G:ft_search_options.py:19-44›

**Example:**
```python
from glide_sync import ft

# NOTE: ft.info(client, ...) NOT client.ft_info(...)
info = ft.info(client, "products_idx")
```

**Reply shape depends on emulate-release.** The `index_definition` block gains `score_field` and a real `default_score` only at `search.emulate-release ≥ 1.3.0`; below that it is 6 elements with `default_score == b"1"` and no `score_field`, even if the index was created with `score_field=` ‹GA:326-329; VS:COMPATIBILITY.md:190; GT:3299›. Look fields up by name, never by position, and treat `score_field` as optional. The FT.INFO format also differs from RediSearch's in general ‹VS:COMPATIBILITY.md:133›.

### ft.aggregate()

**Performs aggregation query.**

```python
ft.aggregate(
    client: GlideClient,
    index_name: str,
    query: str,
    options: Optional[FtAggregateOptions] = None
) -> FtAggregateResponse
```

**Parameters:**
- `client`: GlideClient instance (first parameter, always required)
- `index_name`: Name of the index to aggregate
- `query`: Search query string — **NOT the same as FT.SEARCH**. See wildcard warning below.
- `options`: FtAggregateOptions with LOAD, GROUPBY, REDUCE, SORTBY, etc.

**Match-all query for FT.AGGREGATE is version-dependent**

```python
# search ≥ 1.3 — bare "*" is match-all ‹GT:2500, GT:4026›
ft.aggregate(client, "idx", "*", options)

# search < 1.3 — bare "*" raises RequestError: Invalid query string syntax.
# Use a field filter as match-all:
ft.aggregate(client, "idx", "@price:[0 inf]", options)  # numeric range
ft.aggregate(client, "idx", "@category:{*}", options)    # tag wildcard (if applicable)
```

On search < 1.3 there is no universal match-all for FT.AGGREGATE: use a numeric range `[0 inf]` on any indexed NUMERIC field, or a TAG filter. If the server version is unknown, the numeric-range form works on both.

**Return Format:**
```python
# Returns: List[Mapping[bytes, Any]] — a flat list of row dicts
# There is NO leading integer count (unlike FT.SEARCH and unlike valkey-py).
# Example:
[
    {b'category': b'electronics', b'total': b'1500'},
    {b'category': b'books', b'total': b'320'},
]
```

**LOAD for reducer fields is version-dependent**

- **search < 1.3**: FT.AGGREGATE does not load document fields for REDUCE/SORTBY/APPLY/FILTER. Fields used in SUM, AVG, MIN, MAX, etc. must be explicitly loaded with `loadFields` before the GROUPBY stage. Without LOAD, reducers silently return 0. COUNT is the only exception — it counts rows, not field values.
- **search ≥ 1.3** (#1341, not gated by emulate-release): any declared attribute referenced by GROUPBY, REDUCE, SORTBY, APPLY or FILTER is loaded automatically. **Vector fields are never auto-loaded.** ‹VS:src/commands/ft_aggregate.cc:50-93; GT:2500 (TOLIST/RANDOM_SAMPLE without loadFields)›

An explicit `loadFields` is correct on both versions, so keep it when the server version is unknown or 1.2 must be supported.

**search < 1.3 — Wrong: SUM returns 0 because `price` is not loaded** (on ≥ 1.3 this same code works):
```python
results = ft.aggregate(
    client=client,
    index_name="products_idx",
    query="@price:[0 500]",
    options=FtAggregateOptions(
        clauses=[
            FtAggregateGroupBy(
                ["@category"],
                [FtAggregateReducer("SUM", ["@price"], "total")],
            )
        ]
    ),
)
# Every row has total = '0' on search < 1.3!
```

**Correct on every version — LOAD the field first:**
```python
from glide_sync import ft
from glide_shared.commands.server_modules.ft_options.ft_aggregate_options import (
    FtAggregateOptions,
    FtAggregateGroupBy,
    FtAggregateReducer,
)

results = ft.aggregate(
    client=client,
    index_name="products_idx",
    query="@price:[0 500]",
    options=FtAggregateOptions(
        loadFields=["@price"],  # ← Required for SUM/AVG/MIN/MAX on search < 1.3; harmless on ≥ 1.3
        clauses=[
            FtAggregateGroupBy(
                ["@category"],
                [FtAggregateReducer("SUM", ["@price"], "total")],
            )
        ]
    ),
)
# results: [{b'category': b'electronics', b'total': b'1500'}, ...]
```

Use `loadAll=True` to load all indexed fields (convenient but less efficient).

**Response format differs from valkey-py**

GLIDE's `ft.aggregate()` returns a flat `List[Mapping[bytes, Any]]`. There is no leading
integer count element. This is different from valkey-py which returns `[count, row1, row2, ...]`.

**Wrong — assuming valkey-py format:**
```python
raw = ft.aggregate(client, "idx", "*", options)
total = raw[0]        # This is a dict, not an int!
rows = raw[1:]        # Off by one — you're skipping the first result
```

**Correct — GLIDE format:**
```python
raw = ft.aggregate(client, "idx", "*", options)  # "*" needs search ≥ 1.3
rows = raw            # Flat list of dicts
total = len(raw)      # Count them yourself
```

With `with_cursor=` set, the reply is instead `[rows, cursor_id]` — see [Cursors](#cursors-ftcursor_read--ftcursor_del-search--13).

**FtAggregateOptions — full signature** ‹G:ft_aggregate_options.py:499›:
```python
FtAggregateOptions(
    loadAll=False, loadFields=None,      # camelCase here (FtHybridOptions uses load_all/load_fields)
    timeout=None, params=None, clauses=None,
    verbatim=False, inorder=False, slop=None, dialect=None,
    add_scores=False, scorer=None,       # search ≥ 1.3: ADDSCORES exposes @__score
    with_cursor=None,                    # search ≥ 1.3: FtWithCursor(count=, maxidle=)
)
```
`loadFields` entries are `str`/`bytes` or `LoadField` ‹G:ft_aggregate_options.py:377›.

#### LOAD … AS — `LoadField` (search ≥ 1.3 **and** emulate-release ≥ 1.3.0)

```python
FtAggregateOptions(loadFields=["title", LoadField("price", alias="cost")])
# emulate-release ≥ 1.3.0: rows have b"title" and b"cost"; b"price" is absent
# emulate-release < 1.3.0 (the 1.3 default!) or search 1.2:
#   RequestError: Index: field `AS` does not exist
```
‹G:ft_aggregate_options.py:331-348; GT:2280›. GLIDE does not check this client-side. If the deployment's emulate-release is unknown, load the plain field and rename it in your own code (or with `FtAggregateApply("@price", "cost")`).

#### Reducers

Only three reducers have factories; every other reducer uses the generic constructor `FtAggregateReducer(function, args, name=None)` ‹G:ft_aggregate_options.py:96-224; G:ft_constants.py:121-128›:

| Reducer | Construct | search |
|---|---|---|
| COUNT | `FtAggregateReducer("COUNT", [], "n")` | all |
| COUNT_DISTINCT / SUM / MIN / MAX / AVG / STDDEV | `FtAggregateReducer("SUM", ["@price"], "total")` | all |
| TOLIST (unique values) | `FtAggregateReducer.to_list("@price", name="prices")` | ≥ 1.3 |
| FIRST_VALUE | `FtAggregateReducer.first_value("@title", by="@price", order=OrderBy.DESC, name="top")` | ≥ 1.3 |
| RANDOM_SAMPLE (≤ 1000) | `FtAggregateReducer.random_sample("@price", 10, name="sample")` | ≥ 1.3 |

```python
# Wrong — these factories do not exist
FtAggregateReducer.count()      # AttributeError
FtAggregateReducer.sum("@price")

# Wrong — order without by
FtAggregateReducer.first_value("@title", order=OrderBy.DESC)   # ValueError ‹G:ft_aggregate_options.py:191›
```

TOLIST and RANDOM_SAMPLE return a **list** value in the row (`row[b"prices"]` is `List[bytes]`); TOLIST order is the engine's scan order, not insertion order ‹GT:2500›. Always pass `name=`: the default column name for an unnamed reducer changes with emulate-release (see the [matrix](#server-version-matrix-valkey-search-12-vs-13)).

#### SORTBY and LIMIT

`FtAggregateSortBy` keeps at most `max` rows (default 10). On emulate-release ≥ 1.3.0 an **adjacent** `FtAggregateLimit` (immediately before or after) raises that to offset+count; below 1.3.0, or with any stage between them, it does not ‹G:ft_aggregate_options.py:30-42; GT:2912›:

```python
sort = FtAggregateSortBy([FtAggregateSortProperty("@n", OrderBy.ASC)])

clauses=[sort]                                   # 10 rows (default MAX)
clauses=[sort, FtAggregateLimit(0, 15)]          # 15 at emulate ≥ 1.3.0; 10 below
clauses=[sort, FtAggregateGroupBy(["@n"], []), FtAggregateLimit(0, 15)]  # 10 on both

# Portable: set max explicitly to the page end you need
FtAggregateSortBy([FtAggregateSortProperty("@n", OrderBy.ASC)], max=15)
```

#### Scores in FT.AGGREGATE (search ≥ 1.3)

```python
ft.aggregate(client, "idx", "hello", FtAggregateOptions(
    add_scores=True, loadFields=["title"],
    clauses=[FtAggregateSortBy([FtAggregateSortProperty("@__score", OrderBy.DESC)])]))
# each row has b"__score"
```
‹GT:3496›

### Cursors: ft.cursor_read / ft.cursor_del (search ≥ 1.3)

```python
ft.cursor_read(client, index_name, cursor_id: int, count: Optional[int] = None) -> [rows, cursor_id]
ft.cursor_del(client, index_name, cursor_id: int) -> "OK"
```
‹GA:527,578›

```python
rows, cursor_id = ft.aggregate(client, "idx", "*",
    FtAggregateOptions(loadFields=["n"], with_cursor=FtWithCursor(count=5)))
while cursor_id != 0:
    page, cursor_id = ft.cursor_read(client, "idx", cursor_id)
    rows.extend(page)
```
‹GT:4026›

Rules:
- `with_cursor` changes the reply to `[rows, cursor_id]`; `cursor_id == 0` means exhausted and **no cursor was created** — don't read or delete it ‹VS:docs/commands/ft.aggregate.md:61; GA:542-543›.
- **A cursor lives only on the node that ran the query** ‹VS:docs/commands/ft.cursor.md:12›.
  - **Cluster client (`GlideClusterClient`)**: GLIDE routes FT.AGGREGATE … WITHCURSOR, FT.HYBRID and every FT.CURSOR READ/DEL to the **primary of the index's slot, regardless of `ReadFrom`**, so a `PREFER_REPLICA` / AZ-affinity cluster client still pages correctly ‹glide-core/redis-rs/redis/src/cluster_routing.rs:858-936 @ b7c92e92b; GA:554-555,727-733; GT:3821,4503›.
  - **Standalone client (`GlideClient`) with replicas: NOT pinned.** FT.AGGREGATE is a read-only command, so with `PREFER_REPLICA`/AZ affinity it can open the cursor on a replica, while FT.CURSOR (not read-only) goes to the primary → `Cursor not found` ‹glide-core/src/client/standalone_client.rs:864-873,1023; glide-core/redis-rs/redis/src/cluster_routing.rs:1152 @ b7c92e92b›. Use a `ReadFrom.PRIMARY` client for cursor paging in standalone mode. (Traced in source; no live standalone+replica test exists.)
  - Don't read a cursor through a hand-routed `custom_command`, and expect a primary failover to lose open cursors ‹GA:733-734›.
- Reading a deleted, exhausted or expired cursor raises `RequestError` (`Cursor not found…`) ‹GT:4095›. Cursors expire after `maxidle` ms unread (default 300000).
- FT.HYBRID cursors are read the same way ‹GA:556-558; GT:4503›. `FtSearchOptions` has no `with_cursor` — use `limit=` for FT.SEARCH paging. The server accepts FT.SEARCH WITHCURSOR, but a raw `custom_command` sending it is **not** primary-pinned by GLIDE ‹glide-core/redis-rs/redis/src/cluster_routing.rs:901-903 @ b7c92e92b›.

### ft.hybrid() (search ≥ 1.3)

**Fuses a text/tag/numeric query (SEARCH arm) with a vector KNN (VSIM arm) into one ranking.** Unlike a filtered KNN in FT.SEARCH, both arms contribute to the score ‹VS:docs/commands/ft.hybrid.md:1-7›.

```python
ft.hybrid(
    client, index_name,
    search_arm: FtHybridSearchArm,
    vsim_arm: FtHybridVsimArm,
    options: Optional[FtHybridOptions] = None,
) -> FtAggregateResponse        # List[dict]; [rows, cursor_id] when with_cursor is set
```
‹GA:695›

| Class | Signature |
|---|---|
| `FtHybridSearchArm` | `(query, scorer=None, yield_score_as=None)` ‹G:ft_hybrid_options.py:116› |
| `FtHybridVsimArm` | `(field, vector, k=None, ef_runtime=None, shard_k_ratio=None, yield_score_as=None, filter=None)` ‹G:ft_hybrid_options.py:180› |
| `FtHybridVsimFilter` | `(expression, policy=None, batch_size=None)`; `policy` = `HybridFilterPolicy.ADHOC` \| `.BATCHES` ‹G:ft_hybrid_options.py:20-52› |
| `FtHybridCombine.RRF` / `.Linear` / `.Function` | aliases of `FtHybridRrf(constant, window, yield_score_as)`, `FtHybridLinear(alpha, beta, window, yield_score_as)`, `FtHybridFunction(expr, window, yield_score_as)` ‹G:ft_hybrid_options.py:345,395,458,497-499› |
| `FtHybridOptions` | `(combine=None, load_all=False, load_fields=None, clauses=None, with_cursor=None, local_only=False, timeout=None, params=None, dialect=None)` — **snake_case** `load_*` ‹G:ft_hybrid_options.py:510› |

```python
import struct
from glide_sync import ft, FtHybridSearchArm, FtHybridVsimArm, FtHybridVsimFilter, \
    FtHybridOptions, FtHybridCombine, HybridFilterPolicy, FtAggregateFilter, FtWithCursor

q = struct.pack("2f", 1.0, 1.0)

# Basic: default COMBINE is RRF; rows carry b"__key" and b"__score" ‹GT:4187›
rows = ft.hybrid(client, "idx", FtHybridSearchArm("hello"), FtHybridVsimArm("vec", q, k=5))

# Per-arm scores, LINEAR fusion, post-fusion FILTER clause ‹GT:4219,4277›
rows = ft.hybrid(client, "idx",
    FtHybridSearchArm("hello", yield_score_as="ss"),
    FtHybridVsimArm("vec", q, k=5, yield_score_as="vs"),
    FtHybridOptions(combine=FtHybridCombine.Linear(alpha=0.3, beta=0.7),
                    clauses=[FtAggregateFilter("@ss > 0")]))

# Restrict the vector arm with a filter + execution policy ‹GT:4413›
FtHybridVsimArm("vec", q, k=5, filter=FtHybridVsimFilter("@title:hello", policy=HybridFilterPolicy.BATCHES))
```

Rules:
- **Vector as bytes vs `$name`**: pass raw `bytes` and GLIDE generates the PARAMS entry; pass a **`str`** starting with `$` to reference your own `params` entry. `bytes` are **never** a `$` reference, even if the packed vector happens to start with `0x24` ‹G:ft_hybrid_options.py:244-261; GT:4345›.
  ```python
  FtHybridVsimArm("vec", q, k=1)                                         # bytes → auto PARAMS
  FtHybridVsimArm("vec", "$myvec", k=1), FtHybridOptions(params={"myvec": q})  # str reference
  FtHybridVsimArm("vec", b"$myvec")   # Wrong — sent as a 6-byte raw vector, not a reference
  ```
- A `$name` inside either arm's **query text** is not substituted by the server; only the VSIM vector uses PARAMS ‹VS:docs/commands/ft.hybrid.md:46›.
- **Argument order on the wire** is LOCALONLY → LOAD → TIMEOUT → PARAMS → clauses → WITHCURSOR → DIALECT. `FtHybridOptions.to_args` emits it that way for you; a hand-built `custom_command` that puts an aggregate FILTER right after the VSIM arm is parsed as a VSIM pre-filter and fails ‹G:ft_hybrid_options.py:626-655›.
- **At most 10 rows** come back unless you add an `FtAggregateLimit` clause ‹VS:docs/commands/ft.hybrid.md:59; VS:src/commands/ft_hybrid_parser.cc:116,1063-1079›. `k` defaults to 10 ‹VS:docs/commands/ft.hybrid.md:40; VS:src/commands/ft_hybrid_parser.cc:55,372›.
- A `LOAD` (`load_fields`/`load_all`) **replaces** the implicit `__key`/`__score` columns — add `"@__key"` and `combine=...(yield_score_as="score")` if you need them ‹VS:docs/commands/ft.hybrid.md:55; VS:src/commands/ft_hybrid_parser.cc:995,1040-1045›. `LoadField(..., alias=)` needs emulate-release ≥ 1.3.0, as in FT.AGGREGATE ‹VS:docs/commands/ft.hybrid.md:45›.
- Client-side `ValueError`s: `FtHybridLinear` with only one of alpha/beta; negative RRF `constant`; `batch_size` without `policy=BATCHES`; `ef_runtime` with `policy=ADHOC` ‹G:ft_hybrid_options.py:78,231,367,419›.
- Not supported by the server: VSIM `RANGE` (no typed API; raw command errors), `NOCONTENT`, and FT.HYBRID inside MULTI/EXEC (`Batch(is_atomic=True)`) or Lua ‹VS:docs/commands/ft.hybrid.md:137-141; GT:2837,4701›.
- Cursor paging works as for FT.AGGREGATE: `FtHybridOptions(with_cursor=FtWithCursor(count=...))`, then `ft.cursor_read` ‹GT:4503›.

## Commands the Server Does Not Implement

valkey-search 1.3.0 registers only FT.CREATE, FT.DROPINDEX, FT.INFO, FT._LIST, FT.SEARCH, FT.AGGREGATE, FT.HYBRID, FT.CURSOR (plus FT._DEBUG and FT.INTERNAL_UPDATE) ‹VS:src/module_loader.cc:45-120›. valkey-search 1.2 has none of the alias/explain/profile commands either. These GLIDE functions exist for RediSearch parity but **every call raises `RequestError` (unknown command)** on valkey-search ≤ 1.3 ‹GA:228,255,283,414,443,619,677›:

| Function | Signature (it exists, but the server rejects it) |
|---|---|
| `ft.aliasadd` / `ft.aliasupdate` | `(client, alias, index_name) -> TOK` |
| `ft.aliasdel` | `(client, alias) -> TOK` |
| `ft.aliaslist` | `(client) -> Mapping` |
| `ft.explain` | `(client, index_name, query) -> TEncodable` |
| `ft.explaincli` | `(client, index_name, query) -> List[TEncodable]` |
| `ft.profile` | `(client, index_name, options: FtProfileOptions) -> FtProfileResponse` — `FtProfileOptions.from_query_options(query, FtSearchOptions(...) \| FtAggregateOptions(...))` |

Don't generate code that relies on aliases (e.g. blue/green index swaps via `aliasupdate`), query explanation or profiling against Valkey. Swap indexes in the application instead (e.g. keep the active index name in a key).

## Required Imports

### For ft.create():
```python
# Preferred (v2.3+) — top-level
from glide_sync import (
    ft, FtCreateOptions, TextField, TagField, NumericField,
    VectorField, VectorFieldAttributesFlat, VectorFieldAttributesHnsw,
    VectorAlgorithm, DistanceMetricType, VectorType, DataType,
)

# Legacy (still works)
from glide_sync import ft
from glide_shared.commands.server_modules.ft_options.ft_create_options import (
    FtCreateOptions, Field, TextField, VectorField,
    VectorFieldAttributesFlat, VectorFieldAttributesHnsw,
    VectorAlgorithm, DistanceMetricType, VectorType, DataType,
)
```

### For ft.search():
```python
# Preferred (v2.3+) — top-level
from glide_sync import ft, FtSearchOptions, FtSearchLimit, ReturnField

# Legacy (still works)
from glide_sync import ft
from glide_shared.commands.server_modules.ft_options.ft_search_options import (
    FtSearchOptions, FtSearchLimit, ReturnField,
)
```

### For ft.aggregate():
```python
# Preferred (v2.3+) — top-level
from glide_sync import (
    ft, FtAggregateOptions, FtAggregateGroupBy, FtAggregateReducer,
    FtAggregateFilter, FtAggregateSortBy, FtAggregateSortProperty,
    FtAggregateLimit, FtAggregateApply,
)

# Legacy (still works)
from glide_sync import ft
from glide_shared.commands.server_modules.ft_options.ft_aggregate_options import (
    FtAggregateOptions, FtAggregateGroupBy, FtAggregateReducer,
    FtAggregateFilter, FtAggregateSortBy, FtAggregateSortProperty,
    FtAggregateLimit, FtAggregateApply,
)
```

## Critical Reminders

### THESE DO NOT EXIST:
```python
client.ft_create(...)      # NO - Not a method
client.ft_search(...)      # NO - Not a method
client.ft.create(...)      # NO - No ft attribute
client.ft.search(...)      # NO - No ft attribute
ft.FtCreateOptions(...)    # NO - not part of ft's public API (use: from glide_sync import FtCreateOptions)
```

### Correct PATTERNS:
```python
from glide_sync import ft, FtCreateOptions, FtSearchOptions  # top-level (v2.3+)

ft.create(client, ...)     # YES - Module-level function
ft.search(client, ...)     # YES - Module-level function (positional or keyword args)
ft.dropindex(client, ...)  # YES - Module-level function
FtCreateOptions(...)       # YES - Imported directly, not from ft module
```

## Common Mistakes

1. **Using client methods**: `client.ft_search()` does not exist - use `ft.search(client, ...)`
2. **Using client.ft attribute**: `client.ft.create()` does not exist - use `ft.create(client, ...)`
3. **Wrong FtCreateOptions access**: `ft.FtCreateOptions` is not part of ft's public API — import `FtCreateOptions` directly from `glide_sync`
4. **Not decoding bytes**: Search results return bytes - must decode to strings
5. **Positional args**: Use keyword arguments for clarity
6. **Inferring from Redis-py**: This is NOT Redis-py - do not use Redis-py patterns
7. **Wrong pagination args**: `FtSearchOptions(first_result=0, limit=10)` does not work - use `FtSearchOptions(limit=FtSearchLimit(offset=0, count=10))`
8. **Missing LOAD in FT.AGGREGATE (search < 1.3)**: SUM/AVG/MIN/MAX return 0 without `loadFields` - add `loadFields=["@field"]`. On ≥ 1.3 referenced fields auto-load; explicit LOAD is still portable
9. **Wrong FT.AGGREGATE response format**: GLIDE returns `List[dict]` (flat), not `[count, row1, ...]` like valkey-py; with `with_cursor` it is `[rows, cursor_id]`
10. **Bare `*` on search < 1.3**: raises `Invalid query string syntax` in FT.SEARCH and FT.AGGREGATE - use `@num:[-inf +inf]`; on ≥ 1.3 `*` is fine
11. **`LoadField(alias=)` at default emulate-release**: errors (``field `AS` does not exist``) unless `search.emulate-release ≥ 1.3.0`
12. **`in_keys=[]` meaning "no filter"**: `[]` returns zero results; use `None` for no restriction
13. **Bytes vector as a `$` reference**: in `FtHybridVsimArm`, only a `str` starting with `$` is a reference; `bytes` are always the raw vector
14. **VECTOR_RANGE inside a KNN pre-filter**: rejected on 1.3; so is a second VECTOR_RANGE in one query
15. **Reading a cursor via a replica**: cursors are node-local. A cluster client pins cursor traffic to the slot primary for you; a **standalone** client with `PREFER_REPLICA`/AZ affinity does not — page with a `ReadFrom.PRIMARY` client. Stop at `cursor_id == 0`
16. **Calling alias/explain/profile**: the functions exist but valkey-search ≤ 1.3 rejects the commands
17. **Inventing reducer factories**: only `to_list`, `first_value`, `random_sample` exist; COUNT/SUM/AVG… use `FtAggregateReducer("SUM", ["@f"], "name")`
18. **`loadAll`/`loadFields` vs `load_all`/`load_fields`**: camelCase on `FtAggregateOptions`, snake_case on `FtHybridOptions`
19. **Relying on `unf=True` or `no_highlight=True` to change results**: both are accepted no-ops on 1.3

## See Also

- `python.md` - Full Python guide
- `python-anti-patterns.md` - Common mistakes
