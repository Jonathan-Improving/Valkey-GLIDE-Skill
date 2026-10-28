# Python GLIDE Anti-Patterns

Use when reviewing Python GLIDE code for correctness or debugging runtime failures.

---

## Testing Patterns

### INCORRECT: Mocking at definition location
```python
# Wrong: Mocking at glide_sync module
@patch("glide_sync.GlideClient")  # Won't work if already imported elsewhere
```

### Correct: Mock at import location
```python
# Correct: Mock at the import location
@patch("org.someproject.utilities.valkey.GlideClient")
@patch("org.someproject.utilities.valkey.GlideClusterClient")
def test_something(mock_cluster, mock_client):
    # Mock the create() class method
    mock_client.create.return_value = MagicMock()
    ...
```

**Why:** When code does `from module import get_client`, the name is bound locally. Patching at the source module has no effect on already-imported references.

### INCORRECT: Using MagicMock for async functions
```python
# Wrong: MagicMock for an async function — await hangs indefinitely
with patch("tools.search_manage_index.get_client", return_value=client):
    result = await manage_index(...)  # HANGS — await on non-coroutine
```

### Correct: Use AsyncMock for async functions
```python
from unittest.mock import AsyncMock, patch

# Correct: AsyncMock returns a coroutine that await can resolve
mock = AsyncMock(return_value=client)
with patch("tools.search_manage_index.get_client", mock):
    result = await manage_index(...)  # Works
```

For multiple tool modules that each import the same async function, patch every import location:
```python
mock = AsyncMock(return_value=client)
with (
    patch("tools.search_manage_index.get_client", mock),
    patch("tools.search_add_documents.get_client", mock),
    patch("tools.search_query.get_client", mock),
):
    ...
```

**Why:** `MagicMock.__call__` returns another `MagicMock`, not a coroutine. When you `await` it, the event loop blocks forever. `AsyncMock` returns a proper coroutine.

---

### INCORRECT: Caching GLIDE client across async tests
```python
_client = None

@pytest.fixture()
async def client():
    global _client
    if _client is None:
        _client = await GlideClient.create(config)  # Created on test 1's loop
    yield _client  # Test 2 hangs — different event loop
```

### Correct: Fresh client per test
```python
@pytest.fixture()
async def client():
    c = await GlideClient.create(config)
    yield c
    await c.close()
```

**Why:** A `GlideClient` is bound to the event loop it was created on (Rust FFI/tokio runtime). In pytest with `asyncio_mode = "auto"`, each test function gets its own event loop by default. A client cached from a previous test's loop will hang when used on the current test's loop. The same applies to any async resource cached as a singleton (e.g., `httpx.AsyncClient` inside an embeddings provider).

---

### INCORRECT: Module-scoped async fixtures
```python
@pytest.fixture(scope="module")
async def client():
    c = await GlideClient.create(config)
    yield c
    await c.close()  # Deadlocks — fixture setup blocks the event loop
```

### Correct: Function-scoped async fixtures
```python
@pytest.fixture()
async def client():
    c = await GlideClient.create(config)
    yield c
    await c.close()
```

**Why:** `scope="module"` or `scope="session"` on async fixtures causes deadlocks with `pytest-asyncio` in `auto` mode (at least through version 0.26). The fixture setup blocks the event loop. Use function scope and manage caching yourself if needed.

---

## Query Injection via `=>` Delimiter

The `=>` token in FT.SEARCH syntax separates a filter expression from a KNN vector clause. When user-controlled input is interpolated into query strings without sanitization, an attacker can inject a KNN clause that bypasses all filters and returns all documents. In MCP server contexts, the attacker is an AI agent manipulated via prompt injection.

### VULNERABLE: Interpolating user input into query without sanitization
```python
# DANGEROUS — filter_expression comes from user/agent input
def build_query(filter_expression: str, query_text: str) -> str:
    if filter_expression and query_text.strip() == '*':
        return filter_expression  # ← attacker sends "*=>[KNN 9999 @embedding $vector AS score]"
    return f'({filter_expression}) {query_text}'

# Attacker bypasses year filter, gets ALL documents including secrets
```

### Correct: Reject `=>` in user-supplied input before query construction
```python
# SAFE — reject injection payload before building query
def build_query(filter_expression: str | None, query_text: str) -> str | dict:
    if filter_expression and '=>' in filter_expression:
        return {'status': 'error', 'reason': "filter must not contain '=>'"}
    if filter_expression and query_text.strip() == '*':
        return filter_expression
    if filter_expression:
        return f'({filter_expression}) {query_text}'
    return query_text
```

**Why:** The `=>` delimiter is not documented as security-sensitive and there is no built-in escaping in Valkey Search syntax. Any application that interpolates user input into FT.SEARCH queries must treat `=>` as a reserved token and reject it in user-controlled fragments (filter expressions, vector field names, query text). This applies to all languages, not just Python.

---

## valkey-search 1.3 FT Patterns

Full rules and citations: [python-ft-api.md § Server Version Matrix](python-ft-api.md#server-version-matrix-valkey-search-12-vs-13). Citation legend as in that file.

### INCORRECT: Relying on `LOAD … AS` at the default emulate-release
```python
# search.emulate-release defaults to 1.0.0 even on a 1.3 server
ft.aggregate(client, "idx", "*", FtAggregateOptions(loadFields=[LoadField("price", alias="cost")]))
# RequestError: Index: field `AS` does not exist   ‹GT:2280›
```

### Correct: Rename only where emulate-release ≥ 1.3.0 is guaranteed, else rename client-side
```python
rows = ft.aggregate(client, "idx", "*", FtAggregateOptions(loadFields=["price"]))
rows = [{b"cost": r[b"price"], **{k: v for k, v in r.items() if k != b"price"}} for r in rows]
```

**Why:** `LOAD … AS` is a compatibility fix gated on `search.emulate-release ≥ 1.3.0`; below that `AS` is read as a field name ‹VS:COMPATIBILITY.md:192; VS:src/valkey_search_options.cc:876›.

### INCORRECT: Passing a bytes vector as a `$` reference in FT.HYBRID
```python
FtHybridVsimArm("vec", b"$q", k=5), FtHybridOptions(params={"q": q_bytes})
# b"$q" is sent as a 2-byte raw vector, not a reference to params["q"]
```

### Correct: bytes inline, or a `str` reference plus your own params entry
```python
FtHybridVsimArm("vec", q_bytes, k=5)                                   # GLIDE generates PARAMS
FtHybridVsimArm("vec", "$q", k=5), FtHybridOptions(params={"q": q_bytes})
```

**Why:** `is_param_reference()` is true only for a `str` starting with `$`; a packed vector may legitimately begin with byte `0x24` ‹G:ft_hybrid_options.py:244-261; GT:4345›.

### INCORRECT: Expecting a cursor to be readable on a replica
```python
# Hand-routed cursor read to a replica / any node
client.custom_command(["FT.CURSOR", "READ", "idx", str(cid)], route=RandomNode())
# RequestError: Cursor not found (when it lands on a node other than the one holding the cursor) ‹GA:729-733›
```

### Correct: Use ft.cursor_read and stop at 0
```python
rows, cid = ft.aggregate(client, "idx", "*", FtAggregateOptions(with_cursor=FtWithCursor(count=100)))
while cid != 0:
    page, cid = ft.cursor_read(client, "idx", cid)
    rows.extend(page)
```

**Why:** A cursor exists only on the node that ran the query ‹VS:docs/commands/ft.cursor.md:12›. In **cluster mode** GLIDE routes WITHCURSOR queries and FT.CURSOR to the primary of the index's slot regardless of `ReadFrom` ‹GA:554-555; glide-core/redis-rs/redis/src/cluster_routing.rs:920-936 @ b7c92e92b; GT:3821›. A **standalone** `GlideClient` with `PREFER_REPLICA`/AZ affinity is not pinned: FT.AGGREGATE may open the cursor on a replica while FT.CURSOR goes to the primary — page with a `ReadFrom.PRIMARY` client ‹glide-core/src/client/standalone_client.rs:864-873,1023 @ b7c92e92b›.

### INCORRECT: `in_keys=[]` to mean "no restriction"
```python
FtSearchOptions(in_keys=allowed or [])   # empty → INKEYS 0 → zero results
```

### Correct
```python
FtSearchOptions(in_keys=allowed or None)
```
‹G:ft_search_options.py:173-178,254-258; GT:2432›

### INCORRECT: VECTOR_RANGE as a KNN pre-filter
```python
ft.search(client, "idx", "@vec:[VECTOR_RANGE 5 $b]=>[KNN 3 @vec $b]", FtSearchOptions(params={"b": q}))
# RequestError: VECTOR_RANGE predicates are not supported in the filter of a KNN query
```

### Correct: one or the other
```python
ft.search(client, "idx", "*=>[KNN 3 @vec $b]", FtSearchOptions(params={"b": q}))
ft.search(client, "idx", "@vec:[VECTOR_RANGE 5 $b]=>{$yield_distance_as: d}",
          FtSearchOptions(params={"b": q}, return_fields=[ReturnField("d")]))
```
‹VS:src/query/search.cc:2492-2498; GT:2645›

### INCORRECT: Blue/green index swap with aliases
```python
ft.aliasupdate(client, "products", "products_v2")   # RequestError on valkey-search ≤ 1.3
```

### Correct: Keep the active index name in a key
```python
client.set("products:active_index", "products_v2")
active = client.get("products:active_index").decode()
ft.search(client, active, "@price:[0 100]")
```
‹VS:src/module_loader.cc:45-120; GA:283›

---

## MCP / Server Framework Patterns

GLIDE raises `RequestError` for Valkey errors. These are normal Python exceptions, but in MCP server frameworks (e.g., FastMCP), unhandled exceptions during concurrent tool calls can crash the entire transport (stdio pipe closes, server dies). The crash is in the framework's async dispatch, not in GLIDE's native layer.

**Defensive pattern:** Pre-validate all preconditions before calling GLIDE methods that may raise. Return structured error dicts instead of letting exceptions propagate.

### INCORRECT: Relying on try/except for control flow in MCP tools
```python
# RISKY — RequestError can crash MCP transport
try:
    await ft.info(client, index_name)
except RequestError:
    return {'status': 'error', 'reason': 'Index not found'}
```

### Correct: Pre-validate using safe operations
```python
# SAFE — ft.list() never raises
if not await index_exists(client, index_name):
    return {'status': 'error', 'reason': 'Index not found'}
await ft.info(client, index_name)  # Now safe
```

**Safe operations that never crash:**
- `ft.list(client)` — always returns a list
- `client.exists([key])` — always returns an int
- `client.hget(key, field)` — returns None if missing
- `client.custom_command(['JSON.GET', key, path])` — returns None if missing

**Operations that require pre-validation:**
- `ft.info()`, `ft.create()`, `ft.dropindex()`, `ft.search()` on non-existent index
- `JSON.ARRPOP`, `JSON.ARRTRIM` on non-existent key
- `JSON.ARRAPPEND` on non-array value

---

## JSON Module Patterns

### INCORRECT: Skipping json.dumps for JSON.SET sub-paths
```python
# Wrong — raw string without JSON encoding
await client.custom_command(['JSON.SET', key, '$.name', 'Alice'])
```

### Correct: Always use json.dumps for JSON.SET values
```python
import json

# Correct — json.dumps produces '"Alice"' which is valid JSON
await client.custom_command(['JSON.SET', key, '$.name', json.dumps('Alice')])

# Works for all types and all paths:
await client.custom_command(['JSON.SET', key, '$', json.dumps({"name": "Alice"})])
await client.custom_command(['JSON.SET', key, '$.score', json.dumps(42)])
await client.custom_command(['JSON.SET', key, '$.tags', json.dumps(["a", "b"])])
```

**Why:** `JSON.SET` expects a JSON-encoded value at ALL paths, including sub-paths. If a user reports "double-quoting", the issue is in response parsing, not in how the value is written.

---

### INCORRECT: Calling JSON array ops without checking key/type
```python
# RISKY — crashes if key doesn't exist or path isn't an array
await client.custom_command(['JSON.ARRPOP', key, '$.tags'])
```

### Correct: Pre-validate with JSON.TYPE
```python
async def require_array(client, key: str, path: str) -> dict | None:
    """Returns error dict if not an array, None if OK."""
    exists = await client.exists([key])
    if not exists:
        return {'status': 'error', 'reason': f"Key '{key}' not found"}
    result = await client.custom_command(['JSON.TYPE', key, path])
    jtype = result[0].decode() if isinstance(result, list) else result
    if jtype != 'array':
        return {'status': 'error', 'reason': f"Path is type '{jtype}', not array"}
    return None

# Usage:
err = await require_array(client, key, '$.tags')
if err:
    return err
await client.custom_command(['JSON.ARRPOP', key, '$.tags'])
```

**Why:** `JSON.ARRPOP`, `JSON.ARRTRIM`, and `JSON.ARRAPPEND` raise `RequestError` on non-existent keys or non-array paths, which can crash MCP server transports.

---

## Performance Optimization

### INCORRECT: Fetching entire JSON object
```python
# Inefficient — must fetch/parse entire object
user = json.loads(await client.get("user:123"))
name = user["name"]
```

### Correct: Use JSON.GET with path
```python
# Efficient — fetch only needed fields
name = await client.json_get("user:123", "$.name")
```

**Why:** Fetching only needed fields reduces network transfer and parsing overhead.

---

## Code Patterns

### INCORRECT: Exceptions as Control Flow
```python
async def fetch_data(key: str) -> str:
    value = await client.get(key)
    if value is None:
        raise ValueError("Not found")  # Don't use exceptions for normal logic
    return value
```

### Correct: Status-Based Returns
```python
async def fetch_data(key: str) -> dict:
    value = await client.get(key)
    if value is None:
        return {"status": "error", "msg": "Not found"}
    return {"status": "ok", "data": value}
```

**Why:** Exceptions are expensive and make code hard to read. Use status returns for predictable logic.

---

### INCORRECT: Static Method-Only Classes
```python
class CacheUtils:
    @staticmethod
    async def get_user(client, user_id: str) -> str:
        return await client.get(f"user:{user_id}")
```

### Correct: Module-Level Functions
```python
async def get_user(client, user_id: str) -> str:
    return await client.get(f"user:{user_id}")
```

**Why:** Python has modules for namespacing. Static-only classes add unnecessary boilerplate.

---

### INCORRECT: Tight Coupling
```python
class UserService:
    def __init__(self, host: str, port: int):
        self.config = GlideClientConfiguration([NodeAddress(host, port)])
        self.client = await GlideClient.create(self.config)
```

### Correct: Protocol-Based Abstraction
```python
from typing import Protocol

class CacheClient(Protocol):
    async def get(self, key: str) -> str: ...
    async def set(self, key: str, value: str): ...

class UserService:
    def __init__(self, cache: CacheClient):
        self.cache = cache
```

**Why:** Tight coupling makes testing difficult and prevents swapping implementations.

---

### INCORRECT: Wildcard Imports
```python
from glide import *  # Namespace pollution
```

### Correct: Explicit Imports
```python
from glide import GlideClient, GlideClientConfiguration, NodeAddress
```

**Why:** Wildcard imports pollute namespace, cause name collisions, and break static analysis.

---
