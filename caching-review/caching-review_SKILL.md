---
name: caching-review
description: Detect and report all Spring/JCache caching annotation usage in an application using CAST Imaging. Covers @Cacheable, @CacheEvict, @CachePut, @Caching, @CacheConfig, @Cached (JCache/custom), and reports cache names, eviction gaps, and anti-patterns.
---

# Caching Review

Detect and report all caching annotation usage in an application using the MCP imaging tool. Covers Spring Cache (`@Cacheable`, `@CacheEvict`, `@CachePut`, `@Caching`, `@CacheConfig`) and JCache/custom (`@Cached`, `@CacheResult`, `@CacheRemove`). Reports cache name inventory, eviction coverage, and anti-patterns.

## Steps

1. **Identify the application** — if the user hasn't specified one, call `mcp__imaging__applications` to list available applications and ask which one to review.

2. **Resolve source file paths** — establish the local codebase root, then use it to translate imaging file paths for all `Read` and `Grep` operations.

   **A. Establish `local_root`** — resolve in this order, stopping at the first success:
   1. **Memory:** check memory for a known local root mapped to this application. If found, use it.
   2. **Current directory:** run `Glob **/*.java` from the current working directory. If Java source files are found, treat the current directory as `local_root`.
   3. **Ask the user:** if neither memory nor the current directory contains the codebase, ask: *"Please provide the local filesystem path to the source code root for **\<ApplicationName\>** (e.g. `C:\Users\...\shopizer-main`)."* Confirm with `Glob <path>/**/*.java`. Save the confirmed path to memory under this application name.

   **B. Translate `filePath` values** — for every imaging object path that needs local access:
   | `filePath` format | Action |
   |-------------------|--------|
   | `§{main_sources}§/<relative>` | Replace `§{main_sources}§` with `local_root`. |
   | `§<LISA>§Scr.../<file>` | Decompiled dependency JAR — server-only. **Skip** `Read`/`Grep`; exclude from "File" columns in the report. |
   | Absolute path (`/opt/...`, `C:\...`) | Try `Read` directly. If it fails, join with `local_root` as a fallback. |
   | `null` | No source — skip. |

   > **Test exclusion:** Never read or analyze files under `src/test`, `**/test/**`, or any path containing `/test/`. The review covers production source code only. When using `Grep` or `Glob`, exclude test directories (e.g., `--glob '!**/src/test/**'`). When an MCP result returns a `filePath` containing `/src/test/` or `/test/`, skip it.

### Large Result Caching

   When any MCP call returns more than **200 items** (`metadata.total_items > 200`), do **not** keep the full result in the conversation context. Instead:

   1. Extract only the fields needed for analysis from each item (at minimum: `id`, `name`, `fullname`, `annotations`, `filePath`, `type`).
   2. Write the processed data to a JSON cache file in the current working directory:
      **Filename:** `<app>-caching-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "caching-review",
        "application": "<app-name>",
        "timestamp": "<YYYY-MM-DDTHH:mm:ss>",
        "queries": {
          "<query-label>": {
            "filter": "<filter-string-used>",
            "total_items": N,
            "items": [
              { "id": "...", "name": "...", "fullname": "...", "annotations": ["..."], "filePath": "...", "type": "..." }
            ]
          }
        }
      }
      ```
   3. For subsequent analysis steps, read the cache file with the `Read` tool (using `offset`/`limit` to process in chunks if needed) instead of re-querying the MCP or holding all data in context.
   4. At the end of the report, note: `Data cached to: <filename>`.

   This applies to all paginated MCP calls — `mcp__imaging__objects`, `mcp__imaging__object_details` (large `incoming_calls`), etc. If a result has `metadata.has_next: true`, fetch all pages, merge items, then write the combined result to the cache file.

3. **Detect caching annotations** — run the following queries in parallel via `mcp__imaging__object_details` with `focus="code"`, reading **full content**:

   ### Spring Cache annotations
   - `filters="annotation:contains:Cacheable,type:contains:method"` — `@Cacheable` on methods
   - `filters="annotation:contains:CacheEvict,type:contains:method"` — `@CacheEvict` on methods
   - `filters="annotation:contains:CachePut,type:contains:method"` — `@CachePut` on methods
   - `filters="annotation:contains:Caching,type:contains:method"` — `@Caching` (composite) on methods
   - `filters="annotation:contains:CacheConfig,type:contains:class"` — `@CacheConfig` on classes

   ### JCache / custom annotations
   - `filters="annotation:contains:Cached,type:contains:method"` — `@Cached` on methods (Micronaut, custom, etc.)
   - `filters="annotation:contains:CacheResult,type:contains:method"` — `@CacheResult` (JCache JSR-107)
   - `filters="annotation:contains:CacheRemove,type:contains:method"` — `@CacheRemove` (JCache JSR-107)

   > **Post-filtering required:** `annotation:contains:Cacheable` will also match `@Cacheable` inside a `@Caching(cacheable={...})` composite. Parse result files and filter for exact annotation matches:
   > - `@Cacheable` or `@Cacheable(` — exact Spring cacheable
   > - `@Cached` or `@Cached(` — exact (not `@Cacheable`)
   > - `@CacheEvict` or `@CacheEvict(` — exact eviction
   > - `@CachePut` or `@CachePut(` — exact put
   > - `@Caching` or `@Caching(` — composite annotation
   > - `@CacheConfig` or `@CacheConfig(` — class-level config
   > - `@CacheResult` or `@CacheResult(` — JCache
   > - `@CacheRemove` or `@CacheRemove(` — JCache

4. **Extract cache names** — for each confirmed `@Cacheable`, `@CacheEvict`, `@CachePut`, `@CacheResult`, `@CacheRemove`, and `@CacheConfig` annotation, extract the cache name(s) from the annotation value. Read the source file if annotation text is truncated in Imaging.

   Cache names appear in:
   - `@Cacheable("cacheName")` or `@Cacheable(value = "cacheName")`
   - `@Cacheable(cacheNames = {"name1", "name2"})`
   - `@CacheConfig(cacheNames = {"name1"})`
   - `@CacheResult(cacheName = "name")`

   Build a **cache name inventory**: for each unique cache name, record which methods read from it (`@Cacheable` / `@CacheResult`) and which methods write to / evict from it (`@CacheEvict` / `@CachePut` / `@CacheRemove`).

5. **Classify by architectural layer** — for each annotated method, classify the parent class by layer:

   | Layer | Heuristic |
   |-------|-----------|
   | Service | Class name ends with `Service`, `ServiceImpl`, `Facade`, `FacadeImpl`, or annotations contain `@Service` |
   | Repository | Class name ends with `Repository`, `RepositoryImpl`, `Dao`, `DaoImpl`, or annotations contain `@Repository` |
   | Controller | Class name ends with `Controller`, `RestController`, or annotations contain `@Controller`, `@RestController` |
   | Other | Anything else (utility, helper, etc.) |

   > **Anti-pattern:** `@Cacheable` on `@Repository` methods is a code smell — Spring Data already provides query caching mechanisms, and manual caching at the repository layer bypasses service-level business logic (e.g., filtering, authorization). Flag these.

   > **Anti-pattern:** `@Cacheable` on `@Controller` methods is almost always wrong — controllers should delegate caching to the service layer. Flag these.

6. **Analyze anti-patterns** — for each cache name and each annotated method, check:

   **6a. Cache without eviction (stale data risk):**
   - For each cache name that has `@Cacheable` / `@CacheResult` readers but **no** `@CacheEvict` / `@CacheRemove` writers, flag as a stale data risk.
   - **Why:** Without eviction, cached data grows stale indefinitely. TTL-based eviction (configured externally) may exist, but the code itself has no invalidation path.

   **6b. `@CacheEvict(allEntries = true)` (nuclear eviction):**
   - Read source for each `@CacheEvict` and check for `allEntries = true`.
   - **Why:** Evicts the entire cache on every call, destroying all cached entries. This defeats the purpose of caching for high-throughput caches.

   **6c. `@Cacheable` on void methods:**
   - Check if any `@Cacheable` method returns `void`. Use `source_file_details` inventory to check the method signature, or read the source file.
   - **Why:** Caching a void method makes no sense — there is nothing to cache. This is a bug.

   **6d. `@Cacheable` without a `key` on methods with no parameters:**
   - A method with `@Cacheable` and no parameters uses a default key (empty `SimpleKey`). If multiple such methods share the same cache name, they collide.
   - Check by reading source: if `@Cacheable("X")` has no `key` attribute AND the method has zero parameters, flag if another `@Cacheable("X")` method also has zero parameters.

   **6e. `@Cacheable` and `@CachePut` on the same method:**
   - Flag any method that has both annotations. `@Cacheable` skips execution on cache hit; `@CachePut` always executes. Having both creates confusing semantics.

   **6f. `@Cacheable` on private methods (Spring proxy limitation):**
   - Read source for each `@Cacheable` method. If the method is `private`, flag it.
   - **Why:** Spring Cache uses AOP proxies. Private methods are not proxied, so `@Cacheable` is silently ignored — the method always executes.

   **6g. Self-invocation (same-class calls bypass cache):**
   - Use `mcp__imaging__object_details` with `focus="inward"` on each `@Cacheable` method. If any caller is in the **same class**, flag it.
   - **Why:** Intra-class calls bypass the Spring proxy, so the cache is never consulted. This is the same anti-pattern as `@Transactional` self-invocation.

   **6h. Missing `@CacheConfig` when multiple methods share the same cache name:**
   - If a class has 3+ methods annotated with `@Cacheable`/`@CacheEvict`/`@CachePut` all referencing the same cache name, but the class does not have `@CacheConfig(cacheNames = "...")`, flag as a maintainability issue (cache name duplicated in every annotation).

7. **Check cache configuration** — use `Grep` on `local_root` to find cache infrastructure:

   a. **Cache manager beans:** `Grep` for `CacheManager`, `CaffeineCacheManager`, `RedisCacheManager`, `EhCacheCacheManager`, `ConcurrentMapCacheManager`, `JCacheCacheManager` in `@Configuration` classes.

   b. **Cache properties:** search `application*.yml` and `application*.properties` for:
      - `spring.cache.type` (none / simple / caffeine / redis / ehcache / jcache)
      - `spring.cache.cache-names`
      - `spring.cache.caffeine.spec`
      - `spring.cache.redis.time-to-live`

   c. **`@EnableCaching`:** `Grep` for `@EnableCaching` in `@Configuration` classes. If absent and caching annotations exist, flag as **critical** — all caching annotations are silently ignored without `@EnableCaching`.

8. **Check for cache dependencies in build files** — search `pom.xml` / `build.gradle` for:
   - `spring-boot-starter-cache`
   - `caffeine`
   - `ehcache`
   - `redisson` / `spring-data-redis`
   - `javax.cache` / `jakarta.cache` (JCache API)
   - `hazelcast`

   Cross-reference: if caching annotations exist but no cache dependency is declared, the application uses `ConcurrentMapCacheManager` (Spring's default in-memory fallback) — flag this as a warning for production use.

## Report Template

### Overview
| Metric | Count |
|--------|-------|
| @EnableCaching present | Yes / **No (CRITICAL)** |
| Cache manager type | CaffeineCacheManager / RedisCacheManager / ... / ConcurrentMapCacheManager (default) |
| Cache dependencies | spring-boot-starter-cache, caffeine, ... |
| @Cacheable methods | N |
| @CacheEvict methods | N |
| @CachePut methods | N |
| @Caching (composite) methods | N |
| @CacheConfig classes | N |
| @CacheResult (JCache) methods | N |
| @CacheRemove (JCache) methods | N |
| @Cached (custom/Micronaut) methods | N |
| Unique cache names | N |

### Cache Name Inventory
| Cache Name | @Cacheable | @CacheEvict | @CachePut | Has Eviction? | Status |
|-----------|-----------|------------|----------|--------------|--------|
| cacheName | N methods | N methods | N methods | Yes / **No** | OK / Stale risk |

### Caching by Architectural Layer
| Layer | @Cacheable | @CacheEvict | @CachePut | Assessment |
|-------|-----------|------------|----------|-----------|
| Service | N | N | N | Expected |
| Repository | N | N | N | Anti-pattern if > 0 |
| Controller | N | N | N | Anti-pattern if > 0 |
| Other | N | N | N | — |

### @Cacheable Methods
| Class | Method | Cache Name(s) | Layer | File |
|-------|--------|--------------|-------|------|
| ClassName | methodName | cacheName | Service / Repo / Controller | path/to/File.java |

### @CacheEvict Methods
| Class | Method | Cache Name(s) | allEntries? | Layer | File |
|-------|--------|--------------|------------|-------|------|
| ClassName | methodName | cacheName | Yes / No | Service | path/to/File.java |

### @CachePut Methods
| Class | Method | Cache Name(s) | Layer | File |
|-------|--------|--------------|-------|------|
| ClassName | methodName | cacheName | Service | path/to/File.java |

### Anti-Patterns Found

#### 6a. Cache without eviction
| Cache Name | @Cacheable Methods | Eviction Methods | Risk |
|-----------|-------------------|-----------------|------|
| cacheName | N | 0 | Stale data — no invalidation path in code |

> **Why this matters:** Without explicit eviction, cached data grows stale indefinitely. Even if a TTL is configured externally, there is no programmatic invalidation — meaning a write to the underlying data does not trigger cache invalidation. Callers receive outdated results until the TTL expires.

#### 6b. Nuclear eviction (`allEntries = true`)
| Class | Method | Cache Name | File |
|-------|--------|-----------|------|
| ClassName | methodName | cacheName | path/to/File.java |

> **Why this matters:** `allEntries = true` evicts the **entire cache** on every invocation. For high-throughput caches, this causes a stampede of cache misses and defeats the purpose of caching.

#### 6c. @Cacheable on void methods
| Class | Method | File |
|-------|--------|------|
| ClassName | methodName | path/to/File.java |

> **Why this matters:** A void method has no return value to cache. The `@Cacheable` annotation is silently ignored — this is a bug indicating the developer misunderstood the API.

#### 6d. Key collision risk (zero-parameter methods sharing a cache)
| Cache Name | Methods (all zero-param, no explicit key) |
|-----------|------------------------------------------|
| cacheName | Class.method1(), Class.method2() |

#### 6e. @Cacheable + @CachePut on same method
| Class | Method | File |
|-------|--------|------|
| ClassName | methodName | path/to/File.java |

> **Why this matters:** `@Cacheable` skips execution on cache hit. `@CachePut` always executes and updates the cache. Combining both creates contradictory semantics — the method may or may not execute depending on evaluation order.

#### 6f. @Cacheable on private methods
| Class | Method | File |
|-------|--------|------|
| ClassName | methodName | path/to/File.java |

> **Why this matters:** Spring Cache uses AOP proxies that only intercept public method calls through the proxy. Private methods are called directly, bypassing the proxy entirely — `@Cacheable` is **silently ignored**. The method always executes, and results are never cached.

#### 6g. Self-invocation (intra-class call bypasses cache)
| Class | @Cacheable Method | Called By (same class) | File |
|-------|------------------|----------------------|------|
| ClassName | cachedMethod | callerMethod | path/to/File.java |

> **Why this matters:** Same anti-pattern as `@Transactional` self-invocation. Intra-class calls bypass the Spring AOP proxy, so the cache interceptor is never triggered. The method always executes regardless of cached values.

#### 6h. Missing @CacheConfig (duplicated cache names)
| Class | Methods Sharing Same Cache Name | Cache Name | File |
|-------|-------------------------------|-----------|------|
| ClassName | method1, method2, method3 | cacheName | path/to/File.java |

### Cache Configuration
| Property | Value | Source |
|----------|-------|--------|
| spring.cache.type | caffeine / redis / ... | application.yml |
| @EnableCaching | present / **MISSING** | ConfigClass.java |
| Cache manager bean | CaffeineCacheManager / ... | ConfigClass.java |
| Cache dependency | spring-boot-starter-cache, caffeine | pom.xml |

## Problem Summary

| # | Check | Issues found | Severity |
|---|-------|-------------|----------|
| 1 | @EnableCaching missing | Yes / No | Critical if missing with annotations present |
| 2 | Cache without eviction | N cache(s) | Warning — stale data risk |
| 3 | Nuclear eviction (allEntries=true) | N method(s) | Warning — cache stampede |
| 4 | @Cacheable on void methods | N method(s) | Bug |
| 5 | Key collision risk | N cache(s) | Warning |
| 6 | @Cacheable + @CachePut on same method | N method(s) | Anti-pattern |
| 7 | @Cacheable on private methods | N method(s) | Bug — silently ignored |
| 8 | Self-invocation bypass | N method(s) | Anti-pattern — cache never consulted |
| 9 | Missing @CacheConfig (name duplication) | N class(es) | Maintainability |
| 10 | @Cacheable on @Repository | N method(s) | Code smell |
| 11 | @Cacheable on @Controller | N method(s) | Anti-pattern |
| 12 | No cache library dependency (using ConcurrentMapCacheManager default) | Yes / No | Warning for production |
