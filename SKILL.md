---
name: glide
description: Production-ready patterns for Valkey GLIDE clients across 6 languages. Activate when generating, reviewing, or debugging Valkey GLIDE code including client creation, batch/pipeline/transaction operations, clustering, authentication, TLS, error handling, and timeout configuration.
---

Based on valkey-glide v2.4.0. Detect target version from package manifest before generating code - if the project uses a newer version than v2.4.0, stop and tell the operator this skill needs updating.

## Language Routing

| User task | Open |
|-----------|------|
| Python async/sync, ft.* module functions, batch, vector search, decode bytes | `references/python.md` |
| Python FT module API signatures, ft.create/search/aggregate/dropindex | `references/python-ft-api.md` |
| Python anti-patterns (client.ft_search, redis-py patterns, sequential awaits) | `references/python-anti-patterns.md` |
| Java CompletableFuture, FT static methods, GlideString, batch, Spring | `references/java.md` |
| Java anti-patterns (bare Exception catch, unchecked futures, client methods for FT) | `references/java-anti-patterns.md` |
| Go Result[T], pipeline, CGO, batch pointer semantics, slice params | `references/go.md` |
| Go anti-patterns (standalone/cluster batch mismatch, ignored errors, wrong Del signature) | `references/go-anti-patterns.md` |
| Node.js Promise API, TypeScript, Decoder.Bytes, static FT methods | `references/nodejs.md` |
| Node.js anti-patterns (await on close, missing decoder, client FT methods) | `references/nodejs-anti-patterns.md` |
| PHP C extension, PIE/PECL install, PHPRedis compatibility | `references/php.md` |
| PHP anti-patterns (Composer install attempt, wrong extension name) | `references/php-anti-patterns.md` |
| C# async/await, .NET 8.0+, CustomCommand for FT, await using | `references/csharp.md` |
| C# anti-patterns (sync over async, missing dispose, wrong package) | `references/csharp-anti-patterns.md` |

## Cross-Cutting Routing

| User task | Open |
|-----------|------|
| Client configuration templates (standalone, cluster, auth, TLS) per language | `assets/config-templates.md` |
| Server configuration, cluster sizing, memory policy, ElastiCache | `references/server-configuration-guide.md` |

## Package Selection

| Language | Package | Install |
|----------|---------|---------|
| Python | `valkey-glide` / `valkey-glide-sync` | `pip install valkey-glide` |
| Java | `io.valkey:valkey-glide` | Maven/Gradle with platform classifier |
| Go | `github.com/valkey-io/valkey-glide/go/v2` | `go get` (Go 1.22+) |
| Node.js | `@valkey/valkey-glide` | `npm install` |
| PHP | `valkey_glide` C extension | PECL/pie/source |
| C# | `Valkey.Glide` | `dotnet add package` |

Do not use redis-py, jedis, lettuce, go-redis, StackExchange.Redis, or the `valkey` PyPI package. These are not GLIDE.

## Key Invariants

- FT operations are module-level functions: `ft.create(client, ...)`, `ft.search(client, ...)`, `ft.aggregate(client, ...)`, `ft.dropindex(client, ...)` - not client methods, not custom_command
- Do not use `client.custom_command(["FT.AGGREGATE", ...])` - use `ft.aggregate(client, ...)` instead
- Batch API replaces deprecated Transaction/ClusterTransaction
- Atomic batches require all keys in same hash slot (cluster mode)
- Client is thread-safe and singleton - one per app, auto-reconnects
- Do not recreate client on RequestException/TimeoutError - only on ClosingError
- Request timeout default is 250ms when omitted
- Node.js `client.close()` is synchronous - do not await it
- Go `batch.Del` takes `[]string` (slice), not a single string - passing a single string will not compile
- FT.AGGREGATE rejects wildcard `*` query - use a field filter instead
- FT.AGGREGATE reducers (SUM, AVG, MIN, MAX) require explicit LOAD of their fields - COUNT does not
