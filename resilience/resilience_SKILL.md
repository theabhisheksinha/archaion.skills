---
name: resilience
description: Detect and report resilience anti-patterns in a Spring application using CAST Imaging. Covers resilience4j, Hystrix, Spring Retry dependency detection, annotation inventory, configuration gaps, and 14 anti-pattern checks.
---

# Resilience Review

Detect and report resilience anti-patterns in a Spring application using resilience4j and Spring Retry. Covers dependency detection, annotation inventory (resilience4j + Spring Retry), configuration gaps, and 14 anti-pattern checks.

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

   **Locate configuration files** — once the local codebase root is known, run in parallel:
   - `Glob **/application*.yml` — collect all Spring YAML config files (includes profiles: `application-prod.yml`, etc.)
   - `Glob **/application*.properties` — collect all Spring properties files

   Store the found paths as `config_files`. These are used throughout later checks to verify configuration that is not visible in source code (resilience4j instance config, retry backoff, executor settings, actuator exposure). If none are found, note it in the report and rely on imaging alone for those checks.

### Large Result Caching

   When any MCP call returns more than **200 items** (`metadata.total_items > 200`), do **not** keep the full result in the conversation context. Instead:

   1. Extract only the fields needed for analysis from each item (at minimum: `id`, `name`, `fullname`, `annotations`, `filePath`, `type`).
   2. Write the processed data to a JSON cache file in the current working directory:
      **Filename:** `<app>-resilience-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "resilience",
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

3. **Detect resilience libraries** — run all checks below in parallel:

   **A. Grep pom.xml — resilience4j:**
   Use `Grep` with pattern `resilience4j` on all `pom.xml` files found under the local codebase root. Record every match (groupId, artifactId, version).

   **B. Imaging stubs — resilience4j:**
   Query `mcp__imaging__objects` with `filters="fullname:contains:io.github.resilience4j"`. Also try `filters="fullname:contains:resilience4j"` as a fallback.

   **C. Grep pom.xml — Hystrix:**
   Use `Grep` with pattern `hystrix` (case-insensitive) on all `pom.xml` files. Look for `hystrix-core`, `spring-cloud-starter-netflix-hystrix`, or any `hystrix` artifact. Record groupId, artifactId, version.

   **D. Imaging stubs — Hystrix:**
   Query `mcp__imaging__objects` with `filters="fullname:contains:com.netflix.hystrix"`. Also try `filters="annotation:contains:HystrixCommand"` as a fallback.

   **E. Detect other alternatives** (only if both resilience4j and Hystrix absent):
   - Sentinel: `filters="fullname:contains:com.alibaba.csp.sentinel"`
   - Spring Retry: `filters="annotation:contains:Retryable"` (handled in Check 7 regardless)

   **Decision:**
   - **resilience4j found** → continue with steps 4–10 (Check 4 and sub-checks 5E-5F of Check 5 run), then proceed to step 11. Note in report if Hystrix is also present (migration in progress).
   - **resilience4j NOT found** → skip steps 4–10; jump to step 11 (Check 7). Include a recommendation to adopt resilience4j. Note: Checks 5A-5C (Feign/OpenFeign configuration) still run regardless of resilience4j presence.
   - **Hystrix found** (regardless of resilience4j) → run step 19 (Hystrix checks). Flag Hystrix as **deprecated** — Netflix put Hystrix in maintenance mode in November 2018 and it has received no new features since. Spring Cloud removed Hystrix support in the 2022.x release train (Spring Boot 3.x). Recommend migrating to resilience4j.
   - **Neither found** → report both as absent; note any other alternatives detected (Sentinel, Spring Retry).

4. **Collect resilience annotation inventory** — run all queries below in parallel using `mcp__imaging__objects`, reading **full content**. For each annotation, query both classes and methods:

   | Annotation | Class query | Method query |
   |------------|-------------|--------------|
   | `@CircuitBreaker` | `annotation:contains:CircuitBreaker,type:contains:class` | `annotation:contains:CircuitBreaker,type:contains:method` |
   | `@Retry` | `annotation:contains:Retry,type:contains:class` | `annotation:contains:Retry,type:contains:method` |
   | `@Bulkhead` | `annotation:contains:Bulkhead,type:contains:class` | `annotation:contains:Bulkhead,type:contains:method` |
   | `@RateLimiter` | `annotation:contains:RateLimiter,type:contains:class` | `annotation:contains:RateLimiter,type:contains:method` |
   | `@TimeLimiter` | `annotation:contains:TimeLimiter,type:contains:class` | `annotation:contains:TimeLimiter,type:contains:method` |

   **Post-filter false positives:**
   - `annotation:contains:Retry` also matches `@Retryable` (Spring Retry) — keep only entries where the annotation text starts with `@Retry(` or is exactly `@Retry`. Discard `@Retryable`, `@RetryableTopic`, etc.
   - `annotation:contains:CircuitBreaker` may match Spring Cloud's `@CircuitBreaker` — both are valid resilience patterns; keep all, but note the source in the report if distinguishable from the annotation text.
   - `annotation:contains:RateLimiter` may match Bucket4j or other libs — inspect annotation text for `resilience4j` package reference if ambiguous.

   For each confirmed annotated object, record: `id`, `name`, `fullName`, `type` (class/method), annotation text (to extract attributes like `fallbackMethod`, `name`), `filePath`.

   Build two lookup structures for later checks:
   - `resilience_method_ids`: set of method IDs that carry any resilience annotation
   - `resilience_class_ids`: set of class IDs that carry any resilience annotation at class level

   Paginate all queries if `metadata.has_next: true`.

5. **Check 1 — Missing resilience on outbound HTTP calls** — identify application methods that make outbound HTTP calls but are not protected by any resilience annotation.

   **A. Find HTTP client stub IDs** (in parallel via `mcp__imaging__objects`):
   - `filters="fullname:contains:RestTemplate"`
   - `filters="fullname:contains:WebClient"`
   - `filters="fullname:contains:FeignClient"`
   - `filters="fullname:contains:CloseableHttpClient"`
   - `filters="fullname:contains:java.net.http.HttpClient"`

   Exclude stubs with `§<LISA>§` paths — keep only their `id` values for the next query.

   **B. Find callers** — for each stub ID, call `mcp__imaging__object_details` with `focus=inward` (in parallel). Collect all `incoming_calls` entries that are application code (not stubs). These are the methods making outbound HTTP calls.

   **C. Check resilience coverage** — for each calling method:
   - If its `id` is in `resilience_method_ids` → protected at method level ✓
   - If its parent class ID is in `resilience_class_ids` → protected at class level ✓
   - Otherwise → **unprotected** ⚠

   > **Derive parent class ID:** strip the method name suffix from `fullName` to get the class name, then look it up in the inventory collected in step 4. If not in the inventory, call `mcp__imaging__object_details` with `focus=intra` on the method ID to retrieve its `parent.id`.

   Record each unprotected caller as: `ClassName.methodName → HTTP client used`.

6. **Check 2 — @CircuitBreaker without fallback method** — for each confirmed `@CircuitBreaker` annotation found in step 4, inspect the annotation text:
   - If it contains `fallbackMethod` → protected ✓
   - If it does not → **missing fallback** ⚠

   Record: class/method name, annotation text, file path.

   > Without a `fallbackMethod`, a circuit breaker trip throws a `CallNotPermittedException` directly to the caller with no graceful degradation. The fallback should return a safe default or cached response.

7. **Check 3 — Resilience annotations on wrong layer** — for each confirmed resilience-annotated class or method from step 4, classify its layer using these rules (same as the transactional-review skill):

   **Controller (wrong layer):**
   - Has `@RestController` or `@Controller` annotation, OR
   - Class name contains `Controller` or `Resource` (case-insensitive)

   **Repository / DAO (wrong layer):**
   - Has `@Repository` annotation, OR
   - Class name contains `Repository`, `Dao` (case-insensitive)

   **Service / Facade (correct layer):**
   - Has `@Service`, `@Component`, OR
   - Class name contains `Service`, `Facade`, `Manager`, `Client`, `Gateway`, `Adapter` (case-insensitive)

   Flag any resilience annotation found on a controller or repository.

    > Resilience patterns belong on the **service/facade layer** — the boundary between your application logic and external dependencies. Placing them on controllers mixes HTTP concerns with resilience logic. Placing them on repositories is too deep; a circuit breaker there cannot protect against network failures in the service composition above it.

8. **Check 4 — @Transactional combined with resilience annotation** — 
    > **Run only if resilience4j was detected in step 3.** Skip entirely if resilience4j is absent.

    Cross-reference:

   From the transactional-review approach, find methods/classes that carry both `@Transactional` and any resilience annotation:
   - Run `mcp__imaging__objects` with `filters="annotation:contains:Transactional,annotation:contains:CircuitBreaker,type:contains:method"` (and similar for Retry, Bulkhead, RateLimiter, TimeLimiter) in parallel.
   - Also check for class-level combinations: a class annotated `@Transactional` AND `@CircuitBreaker` (or other).
   - Post-filter: confirm `@Transactional` is real (not `@EnableTransactionManagement`) and resilience annotation is real (same rules as step 4).

   Record: class/method name, the two conflicting annotations, file path.

   > **Why this is dangerous:** `@Retry` (or `@CircuitBreaker` with retry) will re-invoke the method on failure. But if the first attempt already started a `@Transactional` context that rolled back, the retry enters a **new** transaction with no knowledge of the prior state. This can cause:
   > - Duplicate DB writes if the first attempt partially committed before failing.
   > - Phantom reads if the retry sees state written by another transaction between attempts.
   > - Confusing audit trails.
   >
   > **Fix:** Move the `@Transactional` boundary inside the resilience boundary. The resilience annotation goes on an outer method (no transaction); the inner method it calls carries `@Transactional`. This way each retry attempt gets a fresh, clean transaction.

9. **Check 5 — Configuration gaps** — sub-checks 5A-5D run regardless of resilience4j. Sub-checks 5E-5G require resilience4j to be present.

    **A. @FeignClient fallback coverage:**
    Query `mcp__imaging__objects` with `filters="annotation:contains:FeignClient,type:contains:interface"` (repeat for `type=contains:class`). For each result, inspect the annotation text:
    - Contains `fallback=` or `fallbackFactory=` → covered ✓
    - Does not → **missing fallback** ⚠

    Record: interface/class name, full annotation text, file path.

    **B. Feign configuration class — timeout bean:**
    From sub-check A results, extract the `configuration=` value from each annotation text (names the per-client config class). For each named config class:
    - Resolve it via `mcp__imaging__objects` with `filters="fullname:contains:<ClassName>,type:contains:class"`.
    - Call `mcp__imaging__object_details` with `focus=intra` to list its methods.
    - Look for a `@Bean` method whose name or type reference suggests `feign.Request.Options` (common names: `requestOptions`, `feignOptions`, `options`, `timeout`). Alternatively, look for a call to `Request.Options(connectTimeoutMillis, readTimeoutMillis)` in the callee list.
    - If no such bean found → **Feign config class missing `Request.Options` timeout bean** ⚠

    > If a `@FeignClient` has no `configuration=` attribute, it inherits the global `feign.client.config.default.*` properties from application.yml. Note this in the report but do not flag it — the config is external and not verifiable from code alone.

    **C. Feign/OpenFeign native resilience patterns (non-resilience4j):**
    Run these queries to detect native Feign resilience mechanisms:

    1. **Feign Retryer bean** — `mcp__imaging__objects` with `filters="annotation:contains:Bean,fullname:contains:etryer"` (matches `Retryer`, `feignRetryer`, etc.)
       - If found, inspect the bean method for `Retryer.Default` parameters: `interval`, `maxAttempts`, `backoff`
       - Flag if `maxAttempts` ≤ 1 (effectively no retry) or if `interval` > 5000ms without backoff configuration
       
    2. **Feign ErrorDecoder** — `mcp__imaging__objects` with `filters="fullname:contains:ErrorDecoder,type:contains:class"`
       - If found, check if any custom ErrorDecoder implementations exist (not the default `Default`)
       - Custom ErrorDecoders can provide fallback responses without throwing exceptions — this is a form of graceful degradation

    3. **Feign RequestInterceptor for retry** — `mcp__imaging__objects` with `filters="fullname:contains:RequestInterceptor,type:contains:interface"` and `filters="fullname:contains:RetryInterceptor"`
       - Custom interceptors may implement retry logic or header-based fallback injection

    4. **OkHttp / Apache Feign client** — detect if Feign uses OkHttp or Apache as backing client:
       - `filters="fullname:contains:feign.okhttp.OkHttpClient"` → OkHttp-backed Feign
       - `filters="fullname:contains:feign.httpclient.ApacheHttpClient"` → Apache-backed Feign
       - These provide connection pooling and better timeout handling than default

     Record findings: interface/class name, pattern detected, configuration quality notes.

    **D. HTTP client @Bean methods — timeout configuration:**

    Reuse the HTTP client stub IDs from step 5A. For each stub type (RestTemplate, WebClient, etc.), call `mcp__imaging__object_details` with `focus=inward` on the stub class itself to find application methods that instantiate it. Among those callers, retain only methods carrying `@Bean` annotation — these are the factory beans.

    For each `@Bean` factory method, call `mcp__imaging__object_details` with `focus=intra` and examine its outgoing calls. Flag as **missing timeout** ⚠ if none of the callees match these patterns:
    - `setConnectTimeout`, `setReadTimeout` — RestTemplate / `SimpleClientHttpRequestFactory`
    - `connectTimeout`, `readTimeout`, `responseTimeout` — WebClient / Reactor Netty builder
    - `setConnectionTimeout`, `setSoTimeout`, `socketTimeout` — Apache `HttpClient` / OkHttp

    Record: config class, bean method name, client type, file path.

    **E. resilience4j named instances — custom config coverage:**
    > **Run only if resilience4j was detected in step 3.** Skip entirely if resilience4j is absent.

    From the annotation texts collected in step 4, extract every `name=` attribute value (e.g., `"payment"`, `"orderService"`). These are the named resilience4j instances the application declares.

    Query `mcp__imaging__objects` in parallel for `@Bean` methods that produce resilience4j config objects:
    - `filters="annotation:contains:Bean,fullname:contains:CircuitBreakerConfig"`
    - `filters="annotation:contains:Bean,fullname:contains:RetryConfig"`
    - `filters="annotation:contains:Bean,fullname:contains:BulkheadConfig"`
    - `filters="annotation:contains:Bean,fullname:contains:RateLimiterConfig"`
    - `filters="annotation:contains:Bean,fullname:contains:TimeLimiterConfig"`

    If **no config beans found** across all five types: the application relies entirely on resilience4j defaults for all named instances — flag as ⚠ and recommend verifying that `application.yml` defines explicit per-instance configuration (e.g., `resilience4j.circuitbreaker.instances.<name>.*`).
    If **some config beans found**: list them as informational alongside which named instances appear to lack explicit Java-side configuration.

    **F. resilience4j Spring Cloud OpenFeign integration:**
    > **Run only if resilience4j was detected in step 3 AND @FeignClient annotations with resilience4j patterns are found.** Skip if resilience4j is absent.

    Detect if Spring Cloud OpenFeign circuit breaker integration is enabled:
    1. Check for `spring.cloud.openfeign.circuitbreaker.enabled=true` in `config_files`
    2. Check for `@CircuitBreaker` annotations on methods that use `@FeignClient` interfaces
    3. Verify that `@FeignClient` interfaces used with resilience4j have proper fallbackFactory

    If resilience4j CB is enabled but no `@CircuitBreaker` annotations found on Feign client usage → **potential misconfiguration** ℹ

    **G. Native retry coverage — only when resilience4j is absent or unused:**
    > **Skip this sub-check entirely** if step 3 detected resilience4j AND step 4 found at least one annotation. When resilience4j is present and active, this check is not applicable.

    Otherwise, check whether each HTTP call site identified in step 5A has at least one native retry safety net. Run the following queries in parallel:

    1. **Feign `Retryer` beans** — `mcp__imaging__objects` with `filters="annotation:contains:Bean,fullname:contains:etryer"` (matches `Retryer`, `feignRetryer`, `retryer`, etc.). Note: Spring Cloud OpenFeign defaults to `Retryer.NEVER_RETRY`; the absence of a `Retryer` bean means zero automatic retries.

    2. **Spring Retry `@Retryable` on HTTP callers** — `mcp__imaging__objects` with `filters="annotation:contains:Retryable,type:contains:method"`. Cross-reference the result set against the HTTP caller method IDs from step 5A. Matches mean that caller is covered.

    3. **`RetryTemplate` beans** — `mcp__imaging__objects` with `filters="annotation:contains:Bean,fullname:contains:RetryTemplate"`. If found, check which classes use it (`focus=inward`) to determine coverage.

    4. **Apache / OkHttp retry handler** — for each `@Bean` factory method from 5C that produces an Apache `CloseableHttpClient` or OkHttp client, call `mcp__imaging__object_details` with `focus=intra` and look for outgoing calls to `setRetryHandler`, `HttpRequestRetryHandler`, `StandardHttpRequestRetryStrategy`, or OkHttp's `addInterceptor` with a retry interceptor.

    For each HTTP caller from step 5A, classify:
    - Uses a Feign client AND a `Retryer` bean is defined → **covered** ✓
    - Has `@Retryable` on the method → **covered** ✓
    - Caller's class uses a `RetryTemplate` → **covered** ✓
    - Uses Apache/OkHttp client AND retry handler is configured on the bean → **covered** ✓
    - None of the above → **no retry coverage** ⚠

    > **WebClient note:** `.retry()` / `.retryWhen()` are inline reactive operators with no annotation or bean surface — they cannot be detected via static analysis. If WebClient callers are flagged as uncovered, inspect the source files manually before concluding they have no retry.

    Record each uncovered caller as: `ClassName.methodName → HTTP client used`.

10. **Check 6 — @Async methods without @TimeLimiter** — 
    > **Run only if resilience4j was detected in step 3.** Skip entirely if resilience4j is absent. @TimeLimiter is a resilience4j-specific annotation.

    Find all async methods and verify each is time-bounded.

   **A. Find @Async methods:**
   Query `mcp__imaging__objects` with `filters="annotation:contains:Async,type:contains:method"`. Paginate if `metadata.has_next: true`. Record: `id`, `name`, `fullName`, annotation text (to detect return type hints), `filePath`.

   **B. Check @TimeLimiter coverage:**
   For each `@Async` method:
   - If its `id` is in the `resilience_method_ids` set from step 4 **and** the corresponding annotation text contains `TimeLimiter` → **protected** ✓
   - If its parent class ID is in `resilience_class_ids` **and** the class annotation text contains `TimeLimiter` → **protected at class level** ✓
   - Otherwise → **unprotected** ⚠

   To derive the parent class ID: strip the method name suffix from `fullName` to get the class name, look it up in the step 4 inventory. If not found there, call `mcp__imaging__object_details` with `focus=intra` on the method to retrieve its parent.

   **C. Classify return type:**
   Inspect the method's annotation text or use `mcp__imaging__object_details` with `focus=intra` to check the return type:
   - Returns `CompletableFuture`, `Future`, `Mono`, `Flux` → `@TimeLimiter` is **applicable** — flag if missing.
   - Returns `void` → `@TimeLimiter` is **not applicable** (fire-and-forget, no future to cancel). Flag separately with a different recommendation.

   Record each unprotected method as: `ClassName.methodName`, return type, file path.

   > **Why `@Async` without `@TimeLimiter` is risky:**
   > `@Async` offloads execution to a thread pool (`SimpleAsyncTaskExecutor` by default, or a custom `Executor` bean). Without a time limit, a slow external call or lock wait inside the async method holds the worker thread indefinitely. Under sustained load, all threads in the pool block → the pool queue fills → callers that submit new tasks start blocking or being rejected. This is a thread-pool exhaustion pattern that is silent in logs until the pool is full.
   >
   > **Fix for `CompletableFuture` / `Future` return types:** apply `@TimeLimiter(name = "...", cancelRunningFuture = true)` on the method. The limiter will cancel the future and throw `TimeoutException` after the configured duration, freeing the thread.
   >
   > **Fix for `void` return types:** `@TimeLimiter` cannot cancel a fire-and-forget task. Instead:
   > - Configure a bounded `ThreadPoolTaskExecutor` bean with `setKeepAliveSeconds` and `queueCapacity` so stalled tasks don't accumulate indefinitely.
   > - Or move the logic to return a `CompletableFuture` so `@TimeLimiter` can cancel it.

11. **Check 7 — Spring Retry: @Retryable inventory and correctness** — runs regardless of whether resilience4j is present.

   Run in parallel:
   - `mcp__imaging__objects` with `filters="annotation:contains:Retryable,type:contains:method"` — keep only `@Retryable`; discard `@RetryableTopic`, `@RetryableConsumer`, etc.
   - `mcp__imaging__objects` with `filters="annotation:contains:Retryable,type:contains:class"`
   - `mcp__imaging__objects` with `filters="annotation:contains:Recover,type:contains:method"` — recovery methods
   - `mcp__imaging__objects` with `filters="annotation:contains:EnableRetry"` — @EnableRetry declaration
   - Grep `config_files` for `spring.retry` to find any YAML-based Spring Retry config

   Build `retryable_method_ids` and `retryable_class_ids` sets (used by Checks 9–11).

   Sub-checks:
   - **@EnableRetry missing:** if `@Retryable` methods found but no `@EnableRetry` class found in imaging AND no `spring.retry.enabled=true` in config files → @Retryable silently does nothing ⚠
   - **@Retryable without @Recover:** for each `@Retryable` method, check if its parent class has at least one `@Recover` method. Without @Recover, exhausted retries throw the raw exception directly to the caller with no graceful handling.
   - **No backoff configured:** if annotation text contains neither `backoff=` nor a `@Backoff(...)` → fixed 1-second delay (default). Under burst load, simultaneous retries fire at the same instant, creating a retry storm. Cross-check `config_files` for `spring.retry.backoff.*`.

12. **Check 8 — Fallback method existence** — verify that every `fallbackMethod=` reference resolves to a real method.

   For each `@CircuitBreaker` annotation (from step 4) whose text contains `fallbackMethod=`:
   - Extract the value (the method name string).
   - Retrieve the parent class's method list via `mcp__imaging__object_details` with `focus=intra` on the class.
   - If the named method is absent → **fallback not found** ⚠ — a `FallbackMethodNotFoundException` will be thrown at runtime when the circuit opens.
   - If found → additionally Grep the source file for `<returnType>.*<fallbackMethodName>\s*\(` to verify the signature is compatible (same return type, plus optional `Throwable` first parameter).

   Record: annotated class/method, declared `fallbackMethod` name, whether it exists, file path.

13. **Check 9 — Resilience/Retry annotation on private method (AOP bypass)** — AOP proxies do not intercept private methods; the annotation is silently ignored.

   For each method in `resilience_method_ids` ∪ `retryable_method_ids`:
   - Resolve the local file path (step 2 rules). Skip `§<LISA>§` paths.
   - Grep the source file for `\bprivate\b.*\b<methodName>\b\s*\(` (case-sensitive, word boundaries).
   - If matched → **annotation on private method** ⚠

   Record: class name, method name, annotation(s) affected, file path.

14. **Check 10 — Self-invocation bypass** — a Spring bean calling its own annotated method through `this` bypasses the AOP proxy; the annotation is silently ignored.

   For each method in `resilience_method_ids` ∪ `retryable_method_ids`:
   - Call `mcp__imaging__object_details` with `focus=inward` to get the method's callers.
   - Filter callers whose `fullName` shares the same class prefix as the annotated method → these are same-class callers.
   - Flag if any same-class caller found ⚠

   > **Note:** if the class injects itself via `@Autowired MyService self` and calls `self.method()`, the proxy IS used — this pattern is safe. Grep the source file for `@Autowired.*<ClassName>` or `@Resource.*<ClassName>` as a self-injection indicator; if found, downgrade the finding to "informational — verify self-injection is proxy-aware".

15. **Check 11 — @Retry / @Retryable on non-idempotent write operations** — retrying a non-idempotent operation risks duplicate side effects (duplicate orders, duplicate payments, duplicate emails).

   For each method in `resilience_method_ids` (annotation = @Retry) ∪ `retryable_method_ids`:
   - Check annotation text of the *same* method for `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@RequestMapping(method=POST)`, etc.
   - Grep the source file for `@PostMapping\|@PutMapping\|@PatchMapping` within a few lines of the method declaration.
   - Also flag if the method name contains write verbs: `create`, `save`, `insert`, `add`, `submit`, `register`, `place`, `update`, `patch`, `send`, `publish` (case-insensitive).
   - Methods matching either criterion are potentially non-idempotent.

   Record: class, method, annotation(s), reason flagged (mapping annotation or name heuristic), file path.

16. **Check 12 — @Scheduled without resilience** — scheduled tasks calling external services without resilience protection will fail silently and may cascade.

   Query `mcp__imaging__objects` with `filters="annotation:contains:Scheduled,type:contains:method"`. For each found method:
   - If it is in `resilience_method_ids` or its parent class is in `resilience_class_ids` → **protected** ✓
   - Otherwise → call `mcp__imaging__object_details` with `focus=intra` to get outgoing calls. Cross-reference with HTTP client stub IDs from step 5A. If any HTTP client is called → **unprotected scheduled HTTP call** ⚠
   - If no HTTP client call found → note the method exists but is not flagged (no external I/O detected).

17. **Check 13 — Circuit breaker name collision** — two unrelated components sharing a `name=` value share the same CB state; a failure in one trips the breaker for both.

   From step 4 annotation inventory, extract all `name=` attribute values from `@CircuitBreaker` annotations. Build a map: `name → [(className, methodName, filePath)]`.

   Also Grep `config_files` for `resilience4j.circuitbreaker.instances:` to verify which names have explicit YAML config.

   Flag any `name` value that appears in **2+ distinct class names** as a potential collision ⚠ (same name in multiple methods of the *same* class is fine — they intentionally share a breaker).

18. **Check 14 — @Retry + @CircuitBreaker wrong nesting order** — if @Retry is the outer boundary, retries hammer an open circuit causing fast-fail retry spam.

   From step 4, find methods/classes carrying BOTH `@Retry` and `@CircuitBreaker` (or BOTH `@Retryable` and `@CircuitBreaker`):
   - Query `mcp__imaging__objects` with `filters="annotation:contains:Retry,annotation:contains:CircuitBreaker,type:contains:method"` (and class variant).
   - For each found object, Grep the source file for the declaration to read the annotation order as written (top annotation = outer AOP boundary by default in Spring).
   - If `@Retry` / `@Retryable` appears **before** (above) `@CircuitBreaker` in the source → **wrong order** ⚠
   - If `@CircuitBreaker` appears first → correct order ✓

   > **Correct order:**
   > ```java
   > @CircuitBreaker(name = "payment", fallbackMethod = "fallback")  // outer — stops retries when open
   > @Retry(name = "payment")                                         // inner — retries within closed circuit
   > public Response callPayment(...) { ... }
   > ```
   > Wrong order means up to `maxAttempts` calls hit the open circuit per request, each returning a `CallNotPermittedException` immediately — wasting retry budget and polluting logs.

19. **Hystrix inventory and checks** — skip entirely if Hystrix was not detected in step 3.

   **A. Collect @HystrixCommand inventory** — run in parallel:
   - `mcp__imaging__objects` with `filters="annotation:contains:HystrixCommand,type:contains:method"`
   - `mcp__imaging__objects` with `filters="annotation:contains:HystrixCommand,type:contains:class"`
   - `mcp__imaging__objects` with `filters="annotation:contains:HystrixCollapser,type:contains:method"` (request collapsing)
   - `mcp__imaging__objects` with `filters="annotation:contains:EnableHystrix"` (configuration class)
   - `mcp__imaging__objects` with `filters="annotation:contains:EnableCircuitBreaker"` (older activation annotation)

   For each found `@HystrixCommand` object, record: `id`, `name`, `fullName`, `type`, full annotation text (to extract `fallbackMethod`, `commandProperties`, `threadPoolProperties`), `filePath`.

   Build `hystrix_method_ids` and `hystrix_class_ids` for use in sub-checks below.

   **B. @EnableHystrix / @EnableCircuitBreaker missing:**
   If `@HystrixCommand` methods found but neither `@EnableHystrix` nor `@EnableCircuitBreaker` detected → Hystrix AOP is not activated; all `@HystrixCommand` annotations silently do nothing ⚠.

   **C. @HystrixCommand without fallbackMethod:**
   For each `@HystrixCommand` annotation text, check if it contains `fallbackMethod=`.
   - Present → ✓
   - Absent → **missing fallback** ⚠ — when the command fails, the raw exception is thrown to the caller with no graceful degradation.

   **D. @HystrixCommand on wrong layer:**
   Classify each annotated class using the same layer rules as Check 3:
   - Controller (`@RestController`, `@Controller`, name contains `Controller`/`Resource`) → ⚠
   - Repository (`@Repository`, name contains `Repository`/`Dao`) → ⚠
   - Service/Facade/Gateway → ✓

   **E. @HystrixCommand + @Transactional:**
   Query `mcp__imaging__objects` with `filters="annotation:contains:Transactional,annotation:contains:HystrixCommand,type:contains:method"` (and class variant). Apply same post-filter rules as Check 4. Same danger applies: each Hystrix retry or fallback invocation interacts badly with transaction boundaries.

   **F. Fallback method existence:**
   For each `@HystrixCommand` with `fallbackMethod=<name>`, get `focus=intra` on the parent class and verify the named method exists. If absent → `FallbackMethodNotFoundException` at runtime ⚠.
   Also Grep source file for signature compatibility (same return type, `Throwable` optional parameter).

   **G. @HystrixCommand on private method (AOP bypass):**
   For each method in `hystrix_method_ids`, Grep source file for `\bprivate\b.*\b<methodName>\b\s*\(`. If matched → AOP proxy won't intercept it; command is silently ignored ⚠.

   **H. Self-invocation bypass:**
   For each method in `hystrix_method_ids`, call `mcp__imaging__object_details` with `focus=inward`. Filter callers in the same class. Flag if found (same bypass risk as Check 10).

   **I. Execution timeout — commandProperties:**
   For each `@HystrixCommand`, inspect annotation text for `commandProperties=` containing `execution.isolation.thread.timeoutInMilliseconds`. Also Grep `config_files` for `hystrix.command.<commandKey>.execution.isolation.thread.timeoutInMilliseconds`.
   - If no timeout configured: Hystrix default is **1000 ms** — flag as ⚠ if the command calls a slow service (may trip too early) or as informational for fast internal calls.

20. **Produce the report** in this structure:

```
## Resilience Review Report

### Dependency Detection
| Method | Result |
|--------|--------|
| pom.xml Grep | resilience4j found / not found — [matching artifact(s)] |
| Imaging stubs | resilience4j stubs found / not found |

[If not found: note any alternative resilience library detected and recommend adopting resilience4j]

---

### Annotation Inventory
| Annotation | Classes | Methods | Total |
|-----------|---------|---------|-------|
| @CircuitBreaker | N | N | N |
| @Retry | N | N | N |
| @Bulkhead | N | N | N |
| @RateLimiter | N | N | N |
| @TimeLimiter | N | N | N |
| **Total** | **N** | **N** | **N** |

[If none found: resilience4j is on the classpath but no annotations detected — the library is unused.]

---

### Check 1 — Unprotected Outbound HTTP Calls
[If none:] All detected HTTP calls are covered by a resilience annotation — ✓

[If found:]
| Class | Method | HTTP Client | Protection |
|-------|--------|-------------|------------|
| OrderService | createOrder | RestTemplate | None ⚠ |

**Total: N unprotected call site(s) across N class(es)**

> ⚠ **Anti-pattern:** Every outbound HTTP call is a potential failure point. Without a circuit breaker
> or retry, a slow/unavailable downstream service will block threads until the connection timeout,
> causing cascading failure across the application.
>
> **Fix:** Wrap outbound calls with at minimum `@CircuitBreaker(name="...", fallbackMethod="...")`.
> Add `@Retry` for idempotent operations. Use `@TimeLimiter` to enforce a hard timeout.

---

### Check 2 — @CircuitBreaker Without Fallback
[If none:] All @CircuitBreaker annotations define a fallbackMethod — ✓

[If found:]
| Class | Method / Scope | Annotation | File |
|-------|---------------|------------|------|
| PaymentService | processPayment | @CircuitBreaker(name="payment") | path/to/File.java |

**Total: N circuit breaker(s) without fallback**

> ⚠ **Anti-pattern:** A circuit breaker without a fallback throws `CallNotPermittedException`
> directly to the caller when the circuit is open. This provides failure isolation but no graceful
> degradation — the caller still receives an exception.
>
> **Fix:** Add `fallbackMethod = "methodNameFallback"` and implement the fallback returning a safe
> default (cached data, empty collection, or a meaningful error DTO). The fallback method must have
> the same signature as the original plus an optional `Throwable` parameter.

---

### Check 3 — Resilience Annotations on Wrong Layer
[If none:] All resilience annotations are on the service/facade layer — ✓

[If found:]
| Class | Layer | Annotation | File |
|-------|-------|------------|------|
| OrderController | Controller ⚠ | @CircuitBreaker | path/to/File.java |
| UserRepository | Repository ⚠ | @Retry | path/to/File.java |

**Total: N misplaced annotation(s)**

> ⚠ **Anti-pattern:** Resilience patterns belong on the service/facade layer — the boundary
> between business logic and external dependencies. Controllers should delegate to services;
> repositories are internal components, not external integration points.

---

### Check 4 — @Transactional + Resilience Annotation Combined
[If none:] No methods combine @Transactional with a resilience annotation — ✓

[If found:]
| Class | Method | Transactional | Resilience | File |
|-------|--------|--------------|------------|------|
| OrderService | submitOrder | @Transactional | @Retry | path/to/File.java |

**Total: N dangerous combination(s)**

> ⚠ **Anti-pattern — why this is dangerous:**
>
> Attempt 1 opens a transaction, writes rows to the DB, then calls an external service. The call
> fails. Spring rolls back the transaction — the writes are gone. `@Retry` schedules a retry.
>
> Attempt 2 starts a **brand-new transaction from scratch**. The problem is what can happen
> *between* attempt 1's rollback and attempt 2's commit:
>
> 1. **Duplicate writes** — JPA/Hibernate may have flushed rows to the DB before the external call
>    (auto-flush on query, or explicit flush). The DB rolled them back, but Hibernate's first-level
>    cache still holds the entities as "managed". Attempt 2 re-persists them → duplicate insert or
>    silent data corruption if unique constraints are absent.
>
> 2. **Optimistic locking collision** — attempt 1 read an entity at `version = 1`. It rolled back,
>    but another thread incremented the version to `2`. Attempt 2 re-reads `version = 1` from
>    the stale first-level cache and tries to write with `WHERE version = 1` → `OptimisticLockException`.
>    The retry loop then retries *that* exception too, spiralling until max-attempts is exhausted.
>
> 3. **Retry on `DataIntegrityViolationException`** — if the cause of failure is a DB constraint
>    violation (not a transient network error), `@Retry` will retry the exact same operation
>    repeatedly, hitting the same constraint every time. Add `ignoreExceptions` or the method will
>    burn through all retries before throwing.
>
> 4. **`UnexpectedRollbackException`** — if the AOP proxy order puts `@Transactional` *outside*
>    `@Retry` (Spring's default when both are on the same method), the transaction wraps the retry
>    loop. The first failed attempt marks the transaction as rollback-only. The retry runs inside
>    the *same* rolled-back transaction. Spring then throws `UnexpectedRollbackException` on commit
>    — a confusing exception with no clear cause in the logs.
>
> **Fix:** Separate the boundaries. The outer method carries `@Retry` (no `@Transactional`); it
> calls an inner method that carries `@Transactional`. Each retry attempt invokes the inner method
> fresh — it opens a new, clean transaction, commits or rolls back independently, and the retry
> logic never sees a poisoned transaction context.
>
> ```java
> // WRONG — both on the same method
> @Retry(name = "payment")
> @Transactional
> public void submitOrder(...) { ... }
>
> // CORRECT — separated layers
> @Retry(name = "payment")           // outer: no transaction
> public void submitOrder(...) {
>     submitOrderTransactional(...); // inner: fresh TX per attempt
> }
>
> @Transactional
> private void submitOrderTransactional(...) { ... }
> ```

---

### Check 5 — Configuration Gaps

#### 5A — @FeignClient Without Fallback
[If none:] All @FeignClient declarations define `fallback` or `fallbackFactory` — ✓

[If found:]
| Interface / Class | Annotation | File |
|-------------------|------------|------|
| ProductClient | @FeignClient(name="product-service") | path/to/File.java |

**Total: N Feign client(s) without fallback**

> ⚠ **Anti-pattern:** A Feign client without a fallback throws the raw exception to the caller when
> the remote service is unavailable. Add `fallback = ProductClientFallback.class` (implements the
> Feign interface with safe-default methods) or `fallbackFactory = ProductClientFallbackFactory.class`
> (receives the `Throwable` for logging). Requires `spring.cloud.openfeign.circuitbreaker.enabled=true`.

#### 5B — Feign Config Class Without Timeout Bean
[If none or all have Request.Options:] All Feign config classes define a `Request.Options` bean — ✓

[If found:]
| @FeignClient | Config class | Missing | File |
|--------------|-------------|---------|------|
| ProductClient | ProductFeignConfig | Request.Options (timeout) | path/to/File.java |

**Total: N Feign config class(es) without explicit timeout**

> ⚠ **Anti-pattern:** Without a `Request.Options` bean, Feign uses its hardcoded defaults
> (connect: 10 s, read: 60 s). A slow downstream service will hold a thread for up to 60 seconds
> per request. Define an explicit bean:
> ```java
> @Bean
> public Request.Options requestOptions() {
>     return new Request.Options(2000, TimeUnit.MILLISECONDS, 5000, TimeUnit.MILLISECONDS, true);
> }
> ```

#### 5C — HTTP Client @Bean Without Timeout Configuration
[If none:] All HTTP client factory beans configure connection and read timeouts — ✓

[If found:]
| Config class | Bean method | Client type | Missing | File |
|-------------|-------------|-------------|---------|------|
| HttpConfig | restTemplate | RestTemplate | connect/read timeout | path/to/File.java |

**Total: N HTTP client bean(s) without explicit timeout**

> ⚠ **Anti-pattern:** An HTTP client bean without explicit timeouts inherits JVM or OS defaults,
> which are typically very long (minutes) or infinite. Under load this exhausts the thread pool.
> For RestTemplate, configure via `SimpleClientHttpRequestFactory` or `HttpComponentsClientHttpRequestFactory`.
> For WebClient, use `HttpClient.create().responseTimeout(Duration.ofSeconds(5))`.

#### 5D — resilience4j Named Instances Without Custom Configuration
[If custom config beans found:]
The following resilience4j instances have explicit Java-side `@Bean` configuration:
| Config bean | Instance name(s) covered | Type |
|------------|--------------------------|------|
| circuitBreakerConfig | payment, order | CircuitBreakerConfig |

[If no config beans found:]
No custom resilience4j `@Bean` configuration detected. N named instance(s) in use:
`payment`, `orderService`, … — all rely on resilience4j defaults.

> ⚠ **Risk:** resilience4j defaults (50% failure-rate threshold, 5-call sliding window, 3 retries,
> 1 s wait) are conservative stubs, not production-tuned values. Without explicit configuration,
> circuit breakers trip too easily under burst traffic and retry intervals may be too short to allow
> downstream recovery.
>
> **Fix:** Define per-instance configuration either via `@Bean CircuitBreakerConfig` / `RetryConfig` (Java)
> or via `application.yml` `resilience4j.circuitbreaker.instances.<name>.*` properties.
> Verify that every name used in annotations has a matching config block.

#### 5E — Native Retry Coverage (resilience4j absent or unused only)
[Skip and mark N/A if resilience4j was detected and at least one annotation found in step 4]

[If all covered:] All HTTP call sites have at least one native retry mechanism — ✓

[If gaps found:]
| Class | Method | HTTP Client | Retry mechanism |
|-------|--------|-------------|-----------------|
| OrderService | placeOrder | RestTemplate | None ⚠ |
| ProductService | fetchProduct | FeignClient | None ⚠ |

**Total: N call site(s) with no retry coverage**

> ⚠ **Risk:** Without resilience4j or any native retry mechanism, transient failures (network blip,
> brief service restart) propagate immediately as exceptions to the caller. Even a single retry
> dramatically reduces the visible error rate for transient issues.
>
> **Options (pick one per client type):**
> - **Feign:** define a `@Bean Retryer` in the config class —
>   `new Retryer.Default(100, 1000, 3)` (100 ms initial interval, 1 s max, 3 attempts).
> - **RestTemplate:** add `spring-retry` + `@EnableRetry` and annotate the calling method with
>   `@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 200))`.
> - **WebClient (reactive):** use `.retryWhen(Retry.backoff(3, Duration.ofMillis(200)).filter(ex -> isTransient(ex)))`.
>   Note: inline operators — not detectable by static analysis, verify source manually.
> - **Apache HttpClient:** configure `HttpRequestRetryStrategy` on the `HttpClientBuilder`.
>
> ⚠ Only retry **idempotent** operations (GET, PUT, DELETE, or POST with server-side idempotency key).
> Never retry a non-idempotent POST blindly — it risks duplicate writes.

---

### Check 6 — @Async Without @TimeLimiter
[If none:] No @Async methods found, or all are covered by @TimeLimiter — ✓

[If found:]
| Class | Method | Return type | Issue | File |
|-------|--------|-------------|-------|------|
| NotificationService | sendEmail | CompletableFuture | No @TimeLimiter ⚠ | path/to/File.java |
| ReportService | generateReport | void | Fire-and-forget, no pool bound ⚠ | path/to/File.java |

**Total: N async method(s) without time-limit protection (N with applicable @TimeLimiter, N void fire-and-forget)**

> ⚠ **Anti-pattern:** `@Async` without a time limit allows a single slow task to occupy a worker
> thread indefinitely. Under load, all threads in the pool block → the queue fills → callers block
> or get rejected. This is silent thread-pool exhaustion.
>
> - For `CompletableFuture` / `Future` / reactive return types: add `@TimeLimiter(name="...", cancelRunningFuture=true)`.
> - For `void` methods: configure a bounded `ThreadPoolTaskExecutor` with `setKeepAliveSeconds` and
>   `queueCapacity`, or refactor to return `CompletableFuture` so `@TimeLimiter` can cancel it.

---

### Check 7 — Spring Retry (@Retryable) Inventory and Correctness
[If no @Retryable found:] Spring Retry annotations not detected — ✓ (or Spring Retry not used)

**Inventory:**
| Annotation | Classes | Methods | Total |
|-----------|---------|---------|-------|
| @Retryable | N | N | N |
| @Recover | — | N | N |
| @EnableRetry | N | — | N |

**Sub-check results:**

| Issue | Finding |
|-------|---------|
| @EnableRetry present | Yes ✓ / No ⚠ — @Retryable silently does nothing |
| @Retryable without @Recover | N method(s) ⚠ / None ✓ |
| @Retryable without backoff | N method(s) ⚠ / None ✓ |

[If @Retryable without @Recover found:]
| Class | Method | File |
|-------|--------|------|
| EmailService | sendNotification | path/to/File.java |

> ⚠ Without `@Recover`, retries that are exhausted throw the raw exception to the caller. Add a
> `@Recover` method with the same return type and the exception as the first parameter to provide
> graceful degradation.

[If no backoff found:]
> ⚠ Default Spring Retry backoff is a fixed 1-second delay. Under burst load, all in-flight retries
> fire simultaneously at the same instant, creating a retry storm against the downstream service.
> Use `@Retryable(backoff = @Backoff(delay = 200, multiplier = 2, maxDelay = 5000))` for exponential
> backoff with jitter.

---

### Check 8 — Fallback Method Existence
[If none:] All `fallbackMethod=` references resolve to existing methods — ✓

[If found:]
| Class | Annotated method | Declared fallback | Exists | File |
|-------|-----------------|-------------------|--------|------|
| PaymentService | charge | chargeFallback | No ⚠ | path/to/File.java |

**Total: N missing fallback method(s)**

> ⚠ **Runtime crash:** when the circuit opens, resilience4j calls the fallback method by reflection.
> A missing or incompatible method throws `FallbackMethodNotFoundException` — replacing one failure
> mode with another. Ensure the fallback exists in the same class with the same return type and an
> optional `Throwable` as the last parameter.

---

### Check 9 — Resilience / Retry Annotation on Private Method (AOP Bypass)
[If none:] No resilience or retry annotations found on private methods — ✓

[If found:]
| Class | Method | Annotation | File |
|-------|--------|------------|------|
| OrderService | retryOrder | @Retry | path/to/File.java |

**Total: N annotation(s) on private method(s)**

> ⚠ Spring AOP uses JDK dynamic proxies or CGLIB subclass proxies. Neither can intercept `private`
> methods. The annotation is present in source but has zero runtime effect — no retry, no circuit
> breaker, no rate limit. Make the method package-private or `public`, or move the logic to a
> separate bean where the proxy can intercept it.

---

### Check 10 — Self-Invocation Bypass
[If none:] No resilience-annotated methods are called via same-class self-invocation — ✓

[If found:]
| Annotated class | Annotated method | Called by (same class) | Proxy-aware self-injection? | File |
|----------------|-----------------|------------------------|----------------------------|------|
| OrderService | placeOrder | submitOrder | No ⚠ | path/to/File.java |

**Total: N potential self-invocation bypass(es)**

> ⚠ `this.method()` in Java bypasses the Spring AOP proxy — the annotation is silently ignored.
> **Fix options:**
> 1. Inject the bean into itself: `@Autowired private OrderService self;` and call `self.placeOrder()`.
> 2. Move the annotated method to a separate Spring bean.
> 3. Enable AspectJ load-time weaving (heavy-handed but removes the proxy constraint entirely).

---

### Check 11 — @Retry / @Retryable on Non-Idempotent Write Operations
[If none:] No retry annotations found on potentially non-idempotent methods — ✓

[If found:]
| Class | Method | Annotation | Reason flagged | File |
|-------|--------|------------|----------------|------|
| OrderService | createOrder | @Retry | @PostMapping + name heuristic | path/to/File.java |

**Total: N potentially non-idempotent retry target(s)**

> ⚠ Retrying a `POST` / state-mutating operation without server-side idempotency guarantees can cause
> duplicate records, double charges, or duplicate emails. Only retry operations that are safe to
> repeat (GET, idempotent PUT, or POST with a server-checked idempotency key).
> Add `ignoreExceptions` or `exclude` to the annotation to prevent retrying on `DataIntegrityViolationException`.

---

### Check 12 — @Scheduled Without Resilience
[If none:] No unprotected @Scheduled methods calling external services detected — ✓

[If found:]
| Class | Method | HTTP client called | File |
|-------|--------|--------------------|------|
| SyncJob | syncInventory | RestTemplate | path/to/File.java |

**Total: N unprotected scheduled task(s) making external calls**

> ⚠ A scheduled task that calls an external service without resilience protection will fail silently
> (Spring logs the exception and reschedules). If the downstream is slow, the task holds a scheduler
> thread until the connection timeout, which can block subsequent scheduled tasks. Wrap scheduled
> methods with at minimum `@CircuitBreaker` to fast-fail when downstream is unavailable.

---

### Check 13 — Circuit Breaker Name Collision
[If none:] All circuit breaker names are unique across component boundaries — ✓

[If found:]
| Name | Used in | YAML config? |
|------|---------|-------------|
| "default" | OrderService, PaymentService, InventoryService | Yes |

**Total: N name(s) shared across N+ distinct components**

> ⚠ Circuit breaker instances are shared by `name`. A failure in `OrderService` trips the breaker
> named `"default"` — which also blocks `PaymentService` and `InventoryService`. Assign a unique
> descriptive name to each logical dependency boundary (e.g., `"order-service"`, `"payment-gateway"`).

---

### Check 14 — @Retry + @CircuitBreaker Wrong Nesting Order
[If none:] All @Retry + @CircuitBreaker combinations are in the correct nesting order — ✓

[If found:]
| Class | Method | Order detected | Correct order | File |
|-------|--------|---------------|---------------|------|
| PaymentService | charge | @Retry then @CircuitBreaker ⚠ | @CircuitBreaker then @Retry | path/to/File.java |

**Total: N method(s) with wrong annotation order**

> ⚠ Spring AOP applies annotations top-to-bottom (outermost first). With `@Retry` above `@CircuitBreaker`,
> each retry attempt re-enters the CB check. If the circuit is open, every attempt immediately throws
> `CallNotPermittedException` — burning through all retry slots instantly and flooding logs.
> Correct order: `@CircuitBreaker` on top (outer), `@Retry` below (inner).

---

---

### Hystrix — Deprecated Library
[Skip this section if Hystrix was not detected]

> ⚠ **Hystrix is deprecated.** Netflix placed Hystrix in maintenance-only mode in November 2018.
> Spring Cloud removed all Hystrix starters and auto-configuration in the 2022.x release train
> (Spring Boot 3.x). No security patches, no bug fixes, no new features will be released.
> **Migrate to resilience4j** — it is the recommended replacement and is supported by Spring Cloud Circuit Breaker.

**Inventory:**
| Annotation | Classes | Methods | Total |
|-----------|---------|---------|-------|
| @HystrixCommand | N | N | N |
| @HystrixCollapser | — | N | N |
| @EnableHystrix / @EnableCircuitBreaker | N | — | N |

**Checks:**
| Sub-check | Finding | Assessment |
|-----------|---------|------------|
| @EnableHystrix present | Yes / No | ✓ / ⚠ — commands silently inactive |
| @HystrixCommand without fallbackMethod | N | ⚠ / ✓ |
| @HystrixCommand on wrong layer | N | ⚠ / ✓ |
| @HystrixCommand + @Transactional | N | ⚠ / ✓ |
| Fallback method not found | N | ⚠ / ✓ |
| Annotation on private method | N | ⚠ / ✓ |
| Self-invocation bypass | N | ⚠ / ✓ |
| Timeout not configured | N | ⚠ (default 1000 ms) / ✓ |

[If @HystrixCommand without fallbackMethod:]
| Class | Method / Scope | Annotation | File |
|-------|---------------|------------|------|
| PaymentService | charge | @HystrixCommand | path/to/File.java |

[If fallback method not found:]
| Class | Annotated method | Declared fallback | File |
|-------|-----------------|-------------------|------|
| PaymentService | charge | chargeFallback | path/to/File.java |

[If @HystrixCommand on wrong layer:]
| Class | Layer | Annotation | File |
|-------|-------|------------|------|
| OrderController | Controller ⚠ | @HystrixCommand | path/to/File.java |

[If timeout not configured:]
| Class | Method | Timeout configured | File |
|-------|--------|--------------------|------|
| ProductService | fetchProduct | No (default 1000 ms) ⚠ | path/to/File.java |

> ⚠ Hystrix default execution timeout is 1000 ms. This may be too short for some downstream calls
> (causing false circuit trips) or too long for others (excessive thread hold time). Configure
> explicitly via `@HystrixCommand(commandProperties = {@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "2000")})`
> or via `hystrix.command.<commandKey>.execution.isolation.thread.timeoutInMilliseconds` in config.

**Migration path:**
Replace each `@HystrixCommand(fallbackMethod="foo")` with `@CircuitBreaker(name="...", fallbackMethod="foo")` from
`io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker`. Add the `resilience4j-spring-boot2` (or `-spring-boot3`)
starter and define configuration in `application.yml` under `resilience4j.circuitbreaker.instances.<name>.*`.

---

### Summary
| Check | Findings | Assessment |
|-------|---------|------------|
| resilience4j detected | Yes / No | ✓ / ⚠ Not adopted |
| Hystrix detected | Yes / No | ⚠ Deprecated / ✓ Absent |
| resilience4j annotations in use | N total | — |
| Spring Retry (@Retryable) in use | N total | — |
| 1 — Unprotected HTTP calls | N | ⚠ / ✓ |
| 2 — @CircuitBreaker without fallback | N | ⚠ / ✓ |
| 3 — Resilience on wrong layer | N | ⚠ / ✓ |
| 4 — @Transactional + resilience combined | N | ⚠ / ✓ |
| 5A — @FeignClient without fallback | N | ⚠ / ✓ |
| 5B — Feign config without timeout | N | ⚠ / ✓ |
| 5C — HTTP client bean without timeout | N | ⚠ / ✓ |
| 5D — resilience4j instances without custom config | N | ⚠ Review / ✓ |
| 5E — Native retry (no resilience4j) | N / N/A | ⚠ / ✓ / N/A |
| 6 — @Async without @TimeLimiter | N | ⚠ / ✓ |
| 7 — @EnableRetry missing | Yes / No | ⚠ / ✓ / N/A |
| 7 — @Retryable without @Recover | N | ⚠ / ✓ |
| 7 — @Retryable without backoff | N | ⚠ / ✓ |
| 8 — Fallback method not found | N | ⚠ / ✓ |
| 9 — Annotation on private method | N | ⚠ / ✓ |
| 10 — Self-invocation bypass | N | ⚠ / ✓ |
| 11 — Retry on non-idempotent write | N | ⚠ / ✓ |
| 12 — @Scheduled without resilience | N | ⚠ / ✓ |
| 13 — Circuit breaker name collision | N | ⚠ / ✓ |
| 14 — @Retry + @CB wrong order | N | ⚠ / ✓ |
| Hystrix — @EnableHystrix missing | Yes/No/N/A | ⚠ / ✓ / N/A |
| Hystrix — @HystrixCommand without fallback | N / N/A | ⚠ / ✓ / N/A |
| Hystrix — wrong layer | N / N/A | ⚠ / ✓ / N/A |
| Hystrix — @Transactional combined | N / N/A | ⚠ / ✓ / N/A |
| Hystrix — fallback method not found | N / N/A | ⚠ / ✓ / N/A |
| Hystrix — annotation on private method | N / N/A | ⚠ / ✓ / N/A |
| Hystrix — self-invocation | N / N/A | ⚠ / ✓ / N/A |
| Hystrix — timeout not configured | N / N/A | ⚠ / ✓ / N/A |
```

Always report all checks and both annotation inventories. A check with no findings must say "✓ None" rather than be omitted. Check 5E must say "N/A — resilience4j in use" when skipped. Checks 7–14 that involve Spring Retry (7, 9–11, 14) must say "N/A — Spring Retry not detected" when no `@Retryable` is found. All Hystrix rows must say "N/A — Hystrix not detected" when Hystrix is absent.

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | Presence of a resilience library | None / resilience4j / Hystrix / Spring Retry | ⚠ Absent / ℹ Present |
| 2 | Unprotected outbound HTTP calls | N client(s) — no circuit breaker | ⚠ Critical / ✓ None |
| 3 | @CircuitBreaker / @Bulkhead methods without fallback | N method(s) | ⚠ Anti-pattern / ✓ None |
| 4 | @CircuitBreaker / @RateLimiter on Controller layer | N occurrence(s) | ⚠ Wrong layer / ✓ None |
| 5 | @Transactional + @CircuitBreaker on same method | N occurrence(s) | ⚠ Conflict / ✓ None |
| 6 | Configuration gaps (missing timeouts, thresholds) | N missing property/properties | ⚠ / ✓ |
| 7 | @Retryable without backoff / maxAttempts | N method(s) | ⚠ / ✓ None |
| 8 | Hystrix (EOL since 2018) | Present / Absent | ⚠ EOL / ✓ Absent |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | No resilience library | No protection against external failures | Add `resilience4j-spring-boot3` |
| 2 | Unprotected outbound HTTP calls | N client(s) — no circuit breaker | Wrap with @CircuitBreaker + fallback |
| 3 | @Transactional + @CircuitBreaker | N occurrence(s) — semantic conflict | Separate: circuit breaker in outer service, @Transactional in inner service |
| 4 | @CircuitBreaker without fallback | N method(s) — exception propagated without a safety net | Add `fallbackMethod` to each annotation |
| 5 | Hystrix EOL | Present — unmaintained since 2018 | Migrate to resilience4j |
| 6 | @Retryable without backoff | N method(s) — burst retry on failing service | Add `@Backoff(delay=..., multiplier=...)` |
| 7 | Resilience on Controller layer | N occurrence(s) | Move to the service layer |

> Replace N values above with actual counts from the analysis. Remove rows where count is 0 and severity is ✓.

## Optional — Create a Saved View

After producing the report, if any violations were found, offer to create a CAST Imaging saved view per check. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the violating objects?
- Check 1 — Unprotected HTTP calls: N objects
- Check 2 — CircuitBreaker without fallback: N objects
- Check 3 — Wrong layer: N objects
- Check 4 — @Transactional + resilience: N objects
- Check 1 — Unprotected HTTP calls: N objects
- Check 2 — @CircuitBreaker without fallback: N objects
- Check 3 — Wrong layer: N objects
- Check 4 — @Transactional + resilience: N objects
- Check 5A — @FeignClient without fallback: N objects
- Check 5C — HTTP client bean without timeout: N objects
- Check 5E — No retry coverage (if applicable): N objects
- Check 6 — @Async without @TimeLimiter: N objects
- Check 7 — @Retryable without @Recover / no backoff: N objects
- Check 8 — Fallback method not found: N objects
- Check 9 — Annotation on private method: N objects
- Check 10 — Self-invocation bypass: N objects
- Check 11 — Retry on non-idempotent write: N objects
- Check 12 — @Scheduled without resilience: N objects
- Check 13 — CB name collision: N objects
- Check 14 — Wrong @Retry + @CB order: N objects
- Hystrix — @HystrixCommand without fallback: N objects (if applicable)
- Hystrix — wrong layer: N objects (if applicable)
- Hystrix — fallback not found: N objects (if applicable)
- Hystrix — annotation on private method: N objects (if applicable)
Type yes or specify which check(s) to include.
```

> Note: Check 5B (Feign config timeout), 5D (resilience4j defaults), and 13 (name collision) produce class-level findings — include those class IDs in the view if relevant. Check 5E is only offered when resilience4j is absent/unused.

For each check the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="Resilience — <CheckName>"`, and `object_ids` set to the IDs of the violating objects collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
