---
name: error-handling-review
description: Detect and report error handling patterns in a Spring application using CAST Imaging. Covers custom exception inventory (application vs library vs JDK), duplicate exceptions, @ControllerAdvice/@ExceptionHandler usage, generic try-catch patterns, and polymorphic response anti-patterns (Object/String return types).
---

# Error Handling Review

Detect and report error handling patterns in a Spring application using the MCP imaging tool. Inventories custom exceptions (application-written vs library vs JDK), detects duplicate exception classes, analyzes REST API error handling (@ControllerAdvice, @ExceptionHandler, generic try-catch), and flags polymorphic response anti-patterns (returning `Object` or `String` instead of typed DTOs).

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
   | `§<LISA>§Scr.../<file>` | Decompiled dependency JAR — server-only. **Skip** `Read`/`Grep`; mark as "source unavailable" in reports. |
   | Absolute path (`/opt/...`, `C:\...`) | Try `Read` directly. If it fails, join with `local_root` as a fallback. |
   | `null` | No source — skip. |

   > **Test exclusion:** Never read or analyze files under `src/test`, `**/test/**`, or any path containing `/test/`. The review covers production source code only. When using `Grep` or `Glob`, exclude test directories (e.g., `--glob '!**/src/test/**'`). When an MCP result returns a `filePath` containing `/src/test/` or `/test/`, skip it.

### Large Result Caching

   When any MCP call returns more than **200 items** (`metadata.total_items > 200`), do **not** keep the full result in the conversation context. Instead:

   1. Extract only the fields needed for analysis from each item (at minimum: `id`, `name`, `fullname`, `annotations`, `filePath`, `type`).
   2. Write the processed data to a JSON cache file in the current working directory:
      **Filename:** `<app>-error-handling-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "error-handling-review",
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

### Imaging Quality Rules — Prefer Pre-Computed Results

   Several checks in this skill have equivalent **CAST Imaging quality rules** that provide pre-computed static-analysis results. These are far cheaper to query than reading and parsing source files.

   **Principle:** For any check that has a matching quality rule, use a **two-phase approach**:
   - **Phase A (Imaging):** Call `mcp__imaging__quality_insight_violations` with the relevant rule ID. This returns the list of violating objects directly — no source reading needed.
   - **Phase B (Source complement):** If the Imaging rule does not exist for the application, or if additional context is needed (e.g., to distinguish sub-patterns the rule does not differentiate), fall back to `Grep`/`Read` on the source code.

   **Known quality rules relevant to this skill:**

   | Rule ID | Description | Used by |
   |---------|-------------|---------|
   | 7788 | Empty catch blocks | Step 10 (Check #10) |
   | 1060020 | Empty catch blocks in high fan-in methods | Step 10 (Check #10) |
   | 7652 | Exception thrown in catch without chaining cause (CWE-778) | Step 26 (Check #26) |

   > To discover additional rules for a given application, call `mcp__imaging__quality_insights` and search for keywords like "catch", "exception", "error", "throw". New rules found should be noted and used in Phase A for the matching check.

---

## Part 1 — Custom Exception Inventory

3. **Discover all exception classes** — run the following MCP queries in parallel:

   a. `mcp__imaging__objects` with `filters="name:contains:Exception,type:contains:class"` — catches classes with "Exception" in the name.
   b. `mcp__imaging__objects` with `filters="name:contains:Error,type:contains:class"` — catches custom error classes (e.g., `ServiceError`, `ApiError`).

   Paginate if needed (`metadata.has_next`). Merge and deduplicate results by `id`.

4. **Classify each exception** — for each exception class found, classify by origin:

   | Category | Criteria | Example |
   |----------|----------|---------|
   | **Application** (custom) | `filePath` starts with `§{main_sources}§` or resolves to a file under `local_root`. Source is available. | `com.myapp.exception.ProductNotFoundException` |
   | **Library** (third-party, non-JDK) | `filePath` starts with `§<LISA>§` (decompiled JAR) or `external: true`. Source is **not** available locally. Package is NOT `java.*`, `javax.*`, `jakarta.*`, `sun.*`. | `org.springframework.web.client.HttpClientErrorException` |
   | **JDK / Java EE** | Package starts with `java.`, `javax.`, `jakarta.`, `sun.`, or `jdk.`. | `java.lang.IllegalArgumentException`, `javax.persistence.EntityNotFoundException` |

   For application exceptions, also determine:
   - **Parent class**: what does it extend? (`RuntimeException`, `Exception`, a custom base exception, Spring's `ResponseStatusException`, etc.)
   - **Has `@ResponseStatus`?**: check if the class has `@ResponseStatus` annotation (Spring maps it to HTTP status automatically).
   - **HTTP status code**: if `@ResponseStatus(HttpStatus.XXX)` is present, extract the status code.

   Use `mcp__imaging__object_details` with `focus="intra"` on each application exception to get `filePath` and parent class info. For the parent class, check the `outward` focus for inheritance relationships, or read the source file directly.

   > **Efficiency:** Batch by module. Group exceptions by their package prefix, read a representative sample of source files per group, and check `extends` clauses.

5. **Detect duplicate exception classes** — two exceptions are "duplicates" if:

   a. **Same simple name, different packages:** e.g., `com.myapp.catalog.ProductNotFoundException` and `com.myapp.order.ProductNotFoundException`. These cause confusion about which one to catch/throw.

   b. **Same purpose, different names:** e.g., `ResourceNotFoundException`, `EntityNotFoundException`, `ObjectNotFoundException`, `ItemNotFoundException` — all representing "not found" semantics. Group by semantic category:
      - **Not found**: names containing `NotFound`, `Missing`, `NoSuch`, `Unknown` + `Exception`
      - **Already exists**: names containing `AlreadyExists`, `Duplicate`, `Conflict` + `Exception`
      - **Unauthorized / forbidden**: names containing `Unauthorized`, `Forbidden`, `AccessDenied`, `NotAllowed` + `Exception`
      - **Validation / invalid**: names containing `Invalid`, `Validation`, `BadRequest`, `Illegal` + `Exception` (exclude JDK `IllegalArgumentException` etc.)
      - **Service / business**: names containing `Service`, `Business`, `Application` + `Exception`
      - **Generic**: names like `CustomException`, `AppException`, `BaseException`, `GeneralException`

   c. **Unused exceptions:** Use `mcp__imaging__object_details` with `focus="inward"` on each application exception. If `info_incoming_calls` shows 0 callers, the exception is never thrown — flag as dead code.

6. **Analyze exception hierarchy** — build the inheritance tree for application exceptions:

   a. Read source files for all application exceptions and extract the `extends` clause.

   b. Identify the base exception(s): custom exceptions that extend directly from `RuntimeException`, `Exception`, or `ResponseStatusException`.

   c. Build the hierarchy tree. Flag:
      - **Flat hierarchy**: all exceptions extend `RuntimeException` directly — no custom base class. This means no common catch clause can handle "all application exceptions."

        > **Why flat hierarchy is an anti-pattern:**
        > - **No unified catch:** you cannot write `catch (AppBaseException e)` — catching `RuntimeException` also swallows NPEs, `IllegalStateException`, and every JDK/library runtime exception.
        > - **Fragile @ControllerAdvice:** without a common base, you must either list every exception individually in `@ExceptionHandler` (easy to forget new ones) or use `@ExceptionHandler(RuntimeException.class)` which swallows framework exceptions that should propagate (e.g., Spring Security's `AccessDeniedException`).
        > - **No shared fields:** a base class like `AppException(String errorCode, String message)` guarantees consistent fields. Without it, some exceptions carry `errorCode`, some `code`, some nothing — handlers must branch on `instanceof` to extract fields.
        > - **Cross-cutting behavior impossible:** adding a correlation ID, severity level, or user-facing message later requires modifying every exception class instead of adding it once to the base.
        > - **Severity: Medium** — tolerable in small codebases with few exceptions and a well-structured `@ControllerAdvice`; becomes painful as the exception count grows.
      - **Too deep**: hierarchy deeper than 3 levels (e.g., `RuntimeException` → `BaseException` → `ServiceException` → `ProductServiceException` → `ProductNotFoundException`). Over-engineering.
      - **Mixed checked/unchecked**: some application exceptions extend `Exception` (checked) and others extend `RuntimeException` (unchecked). Inconsistent — callers face a mix of mandatory and optional catch clauses.

---

## Part 2 — REST API Error Handling

7. **Detect `@ControllerAdvice` / `@RestControllerAdvice` classes** — run in parallel:

   a. `mcp__imaging__object_details` with `focus="code"`, `filters="annotation:contains:ControllerAdvice,type:contains:class"`
   b. `mcp__imaging__object_details` with `focus="code"`, `filters="annotation:contains:RestControllerAdvice,type:contains:class"`

   Post-filter for exact `@ControllerAdvice` or `@RestControllerAdvice` annotations. Merge and deduplicate.

   For each advice class found, read the source file and extract:
   - All `@ExceptionHandler` methods
   - Which exception types each handler catches (from `@ExceptionHandler(SomeException.class)` or the method parameter type)
   - The return type of each handler method
   - Whether it returns a `ResponseEntity<?>` with a structured error body or a plain string/object

8. **Detect `@ExceptionHandler` methods on controllers** — these are controller-local handlers (not global):

   a. `mcp__imaging__object_details` with `focus="code"`, `filters="annotation:contains:ExceptionHandler,type:contains:method"`

   Post-filter for exact `@ExceptionHandler`. For each, check if the parent class is a `@ControllerAdvice` (already covered in step 7) or a regular `@Controller`/`@RestController` (local handler). Separate the two sets.

   For controller-local handlers, read the source and extract the same info as step 7.

   > **Anti-pattern:** Controller-local `@ExceptionHandler` methods indicate inconsistent error handling — some controllers handle errors differently than others. Flag if there are local handlers AND a global `@ControllerAdvice`.

9. **Analyze exception coverage** — cross-reference:

   a. List all application exceptions (from step 4).
   b. List all exceptions handled by `@ExceptionHandler` methods (from steps 7-8).
   c. Identify **unhandled application exceptions**: custom exceptions that are thrown (have callers from step 5c) but have no `@ExceptionHandler` and no `@ResponseStatus` annotation.

   > **Why:** Unhandled exceptions fall through to Spring's default handler, which returns a 500 with a `DefaultErrorAttributes` JSON body (or a Whitelabel error page). This leaks internal class names and stack traces in non-production profiles.

10. **Detect generic try-catch patterns and empty catch blocks in controllers** — two-phase detection:

    **Phase A — Imaging quality rules (preferred, saves ~60–80% tokens vs source-only):**
    Query `mcp__imaging__quality_insight_violations` for these pre-computed rules:
    - `nature="iso-5055", id="7788"` — **"Avoid empty catch blocks"** — returns all methods with empty catch blocks (catch block contains nothing or only a comment). These are swallowed exceptions (#9).
    - `nature="iso-5055", id="1060020"` — **"Avoid empty catch blocks for methods with high fan-in"** — subset of the above but higher impact (frequently called methods).

    For each violation returned, record the object `id`, `name`, `fullName`, and `filePath`. Cross-reference with the controller class set from step 3 to separate controller-level catches from service-level catches.

    **Phase B — Source-based detection (complements Imaging for non-empty catch blocks):**
    Imaging rules 7788/1060020 only detect **empty** catch blocks. For catch blocks that are non-empty but still problematic (e.g., returns a generic ResponseEntity, logs and swallows), read the source:

    a. For each controller class, `Grep` with `-A 3` context for:
       - `catch (Exception ` or `catch (Throwable ` — generic catch-all
       - `catch (RuntimeException ` — broad catch

    b. For each generic catch found, check what happens inside:
       - **Returns a generic `ResponseEntity`**: e.g., `return ResponseEntity.status(500).body("Internal error")` — manual error response, bypasses `@ControllerAdvice`.
       - **Returns a `Map` or ad-hoc error object**: e.g., `return new HashMap<>()` with error fields — unstructured error response.
       - **Logs and re-throws**: `logger.error(...); throw e;` — acceptable if `@ControllerAdvice` handles it.
       - **Swallows the exception**: catch block with no throw and no meaningful response — data loss risk. (May overlap with Imaging Phase A results — deduplicate.)

    c. Flag each pattern:

    | Pattern | Assessment |
    |---------|-----------|
    | `catch(Exception e)` + generic ResponseEntity | Anti-pattern — bypasses @ControllerAdvice, inconsistent error format |
    | `catch(Exception e)` + swallowed (no throw, no response) | Critical — silent failure, data loss |
    | `catch(Exception e)` + log + re-throw | Acceptable if @ControllerAdvice exists |
    | `catch(SpecificException e)` + typed response | Acceptable but should prefer @ExceptionHandler |

---

## Part 3 — Polymorphic Response Anti-Patterns

11. **Detect handler methods returning `Object` or `String`** — for each controller handler method (annotated with `@GetMapping`, `@PostMapping`, etc.):

    a. Determine the return type. Use `mcp__imaging__source_file_details` with `nature="inventory"` on each controller file — method signatures include the return type (e.g., `createProduct() return org.springframework.http.ResponseEntity`).

    b. Alternatively, read the source file and extract the return type from the method declaration.

    c. Classify return types:

    | Return Type | Assessment |
    |------------|-----------|
    | `ResponseEntity<SpecificDto>` | Good — typed, documented, Swagger-friendly |
    | `ResponseEntity<?>` | Warning — wildcard, type unknown at compile time |
    | `ResponseEntity<Object>` | Anti-pattern — can return any type (DTO on success, error map on failure) |
    | `Object` | Anti-pattern — no type safety, polymorphic response |
    | `String` | Anti-pattern — may return JSON string, error message, or redirect path. Caller cannot distinguish success from failure by type. |
    | `Map<String, Object>` or `Map<String, ?>` | Anti-pattern — unstructured, no schema, not documentable |
    | `void` | Acceptable for 204/DELETE responses |
    | `SpecificDto` (concrete class) | Good — typed |

12. **Detect polymorphic response bodies** — for methods returning `ResponseEntity<?>` or `ResponseEntity<Object>`:

    a. Read the source file and find all `return` statements in the method.
    b. Check if different return paths return different types:
       - Success path: `return ResponseEntity.ok(productDto)`
       - Error path: `return ResponseEntity.badRequest().body(errorMessage)` or `return ResponseEntity.status(500).body(Map.of("error", ...))`

    c. Flag methods where the success body type differs from the error body type — this means the caller must do `instanceof` checks or rely on HTTP status codes alone, and API documentation (Swagger/OpenAPI) cannot represent both schemas for the same endpoint.

    > **Why:** Polymorphic responses break contract-first API design. A client deserializing `ResponseEntity<Object>` gets a raw `LinkedHashMap` (Jackson default) and must manually inspect fields. Swagger/OpenAPI cannot document the response schema. The correct pattern is to always return a typed `ResponseEntity<SpecificDto>` for success and let `@ControllerAdvice` handle errors with a consistent `ErrorResponse` DTO.

13. **Detect `ResponseEntity<String>` for JSON responses** — a specific variant of polymorphism:

    a. Find handler methods returning `ResponseEntity<String>` or `String` with `@ResponseBody`.
    b. Read the source and check if the string is manually constructed JSON: e.g., `"{\"status\":\"ok\"}"` or built via `ObjectMapper.writeValueAsString(...)`.
    c. Flag these — manual JSON string construction bypasses Spring's content negotiation, is error-prone (no escaping guarantees), and cannot be documented in OpenAPI.

---

## Part 4 — Error Response Consistency

14. **Identify error response DTOs** — search for classes that look like standard error response objects:

    a. `Grep` on `local_root` for class names containing: `ErrorResponse`, `ErrorDto`, `ApiError`, `ErrorMessage`, `ExceptionResponse`, `ProblemDetail`.
    b. Also check for Spring's `ProblemDetail` (RFC 7807) usage — `Grep` for `ProblemDetail` in imports.
    c. For each error response class found, read the source and list its fields (e.g., `timestamp`, `status`, `message`, `path`, `errors`, `code`).

15. **Assess error response consistency:**

    a. If multiple error DTOs exist, flag as inconsistency — clients must handle different error shapes depending on which endpoint they call.
    b. If no custom error DTO exists and `@ControllerAdvice` is present, check what it returns — it may use Spring's `DefaultErrorAttributes` (which has a known shape but is not a custom DTO).
    c. If `@ControllerAdvice` returns different types for different exceptions, flag — the error response shape should be consistent regardless of the exception type.

    > **Best practice:** A single `ErrorResponse` DTO used across all `@ExceptionHandler` methods, ideally following RFC 7807 (`ProblemDetail`): `type`, `title`, `status`, `detail`, `instance`, plus optional extension fields.

16. **RFC 7807 compliance gap analysis** — for each error response DTO found in step 14:

    a. Compare its fields against the RFC 7807 standard fields:

    | RFC 7807 field | Type | Required? | Purpose |
    |----------------|------|-----------|---------|
    | `type` | URI | Yes | Machine-readable error type identifier (e.g., `https://api.example.com/errors/not-found`) |
    | `title` | String | Yes | Human-readable summary (e.g., "Resource not found") |
    | `status` | int | Yes | HTTP status code as integer (e.g., `404`) |
    | `detail` | String | Recommended | Human-readable explanation specific to this occurrence |
    | `instance` | URI | Optional | URI identifying this specific error occurrence (for correlation) |

    b. For each DTO, produce a gap table: which RFC 7807 fields are present, which are missing, and which have incorrect types (e.g., `status` is `String` instead of `int`, or `errorCode` is `String` instead of an HTTP status integer).

    c. Check if the application uses Spring 6+ `ProblemDetail` class — `Grep` for `import org.springframework.http.ProblemDetail`. If Spring 6+ is available but `ProblemDetail` is not used, flag as a missed opportunity.

    > **Why:** RFC 7807 (`application/problem+json`) is the IETF standard for HTTP API error responses. Spring 6+ provides native `ProblemDetail` support. Custom error DTOs that diverge from this standard force clients to handle non-standard error shapes per API provider.

17. **Error code type mismatch** — for each error response DTO:

    a. If a field represents the HTTP status or error code (commonly named `errorCode`, `code`, `status`, `statusCode`), check its declared Java type.
    b. Flag if the type is `String` when it should be `int`/`Integer` — a String error code like `"500"` requires parsing to compare with HTTP status codes, and risks holding non-numeric values (e.g., `"INTERNAL_ERROR"`) that clients must handle differently.
    c. Also flag if the application exception hierarchy uses `String errorCode` while `@ExceptionHandler` methods set numeric HTTP statuses — the mismatch between the internal code representation and the external HTTP status creates confusion.

    > **Why:** HTTP status codes are integers by specification (RFC 7231). Storing them as `String` in error DTOs or exception fields invites type confusion and parsing errors. If the error code is a business-level code (not HTTP status), it should be a separate field from the HTTP `status`.

18. **Internal detail leakage in error messages** — for each `@ExceptionHandler` method and generic catch block:

    a. Read the source and check what goes into the error response body's message field. Flag these patterns:
       - `exception.getMessage()` passed directly as the response message — may contain internal class names, SQL fragments, file paths, or stack trace elements.
       - `exception.getCause().getMessage()` — exposes root cause details (e.g., JDBC driver errors, ORM exceptions).
       - `exception.toString()` — includes the fully qualified class name + message.
       - `e.getStackTrace()` or `ExceptionUtils.getStackTraceAsString(e)` — full stack trace in response body.

    b. The safe pattern is to return a **generic user-facing message** (e.g., "An internal error occurred") and log the full exception server-side:
       ```java
       logger.error("Unexpected error", exception);
       return new ErrorResponse(500, "An internal error occurred. Reference: " + correlationId);
       ```

    > **Why:** Exposing `exception.getMessage()` to API clients leaks internal implementation details — database column names, ORM entity names, file system paths, and third-party library versions. This is an information disclosure vulnerability (OWASP A01:2021).

19. **NPE risk in error handlers** — for each `@ExceptionHandler` method and catch block:

    a. Read the source and check for unsafe null access patterns:
       - `exception.getCause()` used without null check — not all exceptions have a cause (returns `null`).
       - `Objects.requireNonNull(exception.getCause())` — throws `NullPointerException` in the error handler itself, causing a 500 from the error handler that was supposed to return a meaningful error.
       - `exception.getCause().getMessage()` without guarding against `getCause() == null`.

    b. Flag severity as **Critical** — an NPE in an error handler turns a handled exception into an unhandled `NullPointerException`, which either falls through to a generic 500 or causes a recursive handler invocation.

    > **Why:** Error handlers are the last line of defense. If they throw, the application returns Spring's default error response with potential stack trace leakage. `getCause()` is nullable by contract (`Throwable.getCause()` returns `null` if no cause was set), so any error handler that calls it without a null guard is a ticking time bomb.

20. **Parallel exception hierarchies** — check if the application has multiple, disconnected exception base classes:

    a. From the hierarchy built in step 6, identify all distinct root base classes (excluding JDK/library exceptions). For example:
       - `GenericRuntimeException extends RuntimeException` (unchecked tree)
       - `ServiceException extends Exception` (checked tree)

    b. For each pair of parallel hierarchies, compare their field structures:
       - Do they carry the same fields (e.g., both have `errorCode` + `message`)?
       - Are the field types consistent (e.g., `errorCode` is `String` in one and `int` in another)?
       - Does the checked hierarchy have fields that the unchecked one lacks, or vice versa?

    c. Flag divergent field structures — this means catch handlers must handle different exception shapes depending on whether they catch the checked or unchecked variant. This is especially problematic when a `@ExceptionHandler` catches the base class of one hierarchy but not the other.

    > **Why:** Parallel exception hierarchies with different field structures create a fractured error model. Developers must remember which hierarchy to use, catch handlers are written against one but not the other, and the error response construction logic must branch on exception type. Consolidate into a single hierarchy (prefer unchecked) with consistent fields.

21. **Missing @ExceptionHandler for thrown application exceptions** — cross-reference:

    a. From step 5c, list all application exceptions that have callers > 0 (i.e., are actively thrown in the codebase).
    b. From steps 7-8, list all exceptions covered by `@ExceptionHandler` methods.
    c. From step 4, list all exceptions with `@ResponseStatus` annotation.
    d. Any exception in (a) that is NOT in (b) and NOT in (c) is **unhandled** — it will fall through to Spring's default `BasicErrorController`, which returns `DefaultErrorAttributes` (including stack traces in non-production).

    e. Specifically check for these commonly missed exceptions:
       - `ConstraintViolationException` (Bean Validation on path/query params) — often thrown but no handler defined, resulting in a 500 instead of 400.
       - `MethodArgumentNotValidException` (Bean Validation on `@RequestBody`) — Spring returns 400 by default but with `DefaultErrorAttributes` format, not the app's custom error DTO.
       - Application-specific exceptions that have dedicated subclasses but only the subclasses have handlers (the parent exception has no handler — if thrown directly, it's unhandled).

    > **Why:** Every actively thrown exception without a handler or `@ResponseStatus` is a potential 500 with stack trace leakage. This is especially dangerous for validation exceptions (`ConstraintViolationException`) that should return 400 but default to 500.

22. **@ControllerAdvice scope limitation** — for each `@ControllerAdvice` class:

    a. Check if the annotation has scope-limiting attributes:
       - `@ControllerAdvice(basePackages = "com.myapp.api")` — only applies to controllers in that package.
       - `@ControllerAdvice(assignableTypes = {FooController.class})` — only applies to specific controllers.
       - `@ControllerAdvice(annotations = RestController.class)` — applies to controllers with a specific annotation.

    b. If scoped, check if all controllers in the application fall within that scope. Use `mcp__imaging__objects` with `filters="annotation:contains:Controller"` to list all controllers, then verify each controller's package or type matches the advice scope.

    c. Flag any controllers that are **outside the scope** — they will not benefit from the `@ControllerAdvice` error handling and will fall through to Spring's default error handling.

    > **Why:** A scoped `@ControllerAdvice` is a silent trap. Developers assume all controllers are covered, but controllers in other packages silently fall back to `DefaultErrorAttributes`. This is especially dangerous when new modules or packages are added — the new controllers are unprotected unless someone remembers to update the advice scope.

23. **Service methods returning error DTOs (layer violation)** — for each `@Service` class:

    a. Identify error response DTOs from step 14 (e.g., `ErrorEntity`, `ErrorResponse`, `ApiError`, `ProblemDetail`). Also include `ResponseEntity` — a service should never return an HTTP response wrapper.

    b. Use `mcp__imaging__source_file_details` with `nature="inventory"` on each service class to get method signatures with return types. Alternatively, `Grep` the source files for method declarations.

    c. Flag any service method whose return type is:
       - An error DTO class (from step 14)
       - `ResponseEntity` or `ResponseEntity<...>` (HTTP concern in service layer)
       - `Object` used to return either a success DTO or an error DTO depending on the outcome

    d. For flagged methods, read the source to confirm the method constructs or returns an error DTO on failure paths instead of throwing an exception.

    | Return Type | Assessment |
    |------------|-----------|
    | `ErrorResponse` / `ErrorEntity` / etc. | Anti-pattern — service knows about error presentation |
    | `ResponseEntity<?>` | Anti-pattern — HTTP concern leaked into service layer |
    | `Object` (returns DTO or error) | Anti-pattern — polymorphic return to carry errors |
    | Throws exception on failure | Correct — let @ControllerAdvice convert to error response |

    > **Why:** The service layer should be HTTP-agnostic. When a service returns an error DTO, it:
    > - **Violates separation of concerns** — the service becomes aware of how errors are presented to HTTP clients, coupling business logic to the transport layer.
    > - **Bypasses @ControllerAdvice** — errors packaged as return values cannot be intercepted by exception handlers, so the centralized error handling is short-circuited.
    > - **Forces callers to inspect return values** — every controller method must check `if (result instanceof ErrorResponse)` instead of relying on the clean exception propagation path.
    > - **Breaks reusability** — if the service is called from a message listener, a scheduled task, or another service, the caller receives an HTTP-shaped error DTO that makes no sense outside the web layer.
    > - **Fix:** Replace error DTO returns with domain exceptions (e.g., `throw new ProductNotFoundException(id)`). Let the exception propagate to `@ControllerAdvice`, which converts it to the error response DTO. The service stays HTTP-agnostic.

24. **Exceptions carrying HTTP status codes as runtime constructor parameters** — for each application exception (from step 4):

    a. Read the source file and inspect the constructor(s). Flag exceptions where:
       - A constructor parameter represents an HTTP status code passed at throw-time: e.g., `new ServiceException("Not found", 404)`, `new AppException(message, HttpStatus.NOT_FOUND)`, `new GenericException(errorCode, message)` where `errorCode` is a String/int holding an HTTP status.
       - The exception has a field like `httpStatus`, `statusCode`, `errorCode` (int or String) that callers populate with HTTP status values at throw sites in service classes.

    b. Distinguish from the **acceptable** pattern — HTTP status declared at the class level:
       - `@ResponseStatus(HttpStatus.NOT_FOUND)` on the exception class — compile-time, declarative, not decided by the service.
       - `extends ResponseStatusException` with the status passed in the constructor — borderline but Spring-endorsed; acceptable when the exception itself is domain-specific (e.g., `ProductNotFoundException extends ResponseStatusException`). Anti-pattern when it's a generic exception used for all statuses (e.g., `throw new ResponseStatusException(HttpStatus.CONFLICT, "...")` directly from service code).

    c. Classification:

    | Pattern | Assessment |
    |---------|-----------|
    | `@ResponseStatus(NOT_FOUND)` on exception class | Acceptable — compile-time mapping, service doesn't choose the code |
    | `extends ResponseStatusException` (domain-specific subclass) | Acceptable — Spring-endorsed, status is part of the exception's identity |
    | `new ServiceException(message, 404)` thrown from service | Anti-pattern — service decides HTTP status at runtime |
    | `new GenericException(errorCode, message)` with HTTP int/String | Anti-pattern — one generic exception replaces a meaningful hierarchy |
    | `throw new ResponseStatusException(HttpStatus.XXX, ...)` from service | Anti-pattern — service directly uses HTTP semantics |

    d. For anti-pattern cases, also check the throw sites: `Grep` on `local_root` for `throw new <ExceptionName>` to see if service classes pass different HTTP codes at different call sites. Multiple distinct HTTP codes passed to the same exception class confirms the anti-pattern.

    > **Why exceptions should NOT carry runtime HTTP codes:**
    > - **Service makes an HTTP decision** — choosing 404 vs 409 vs 429 is a presentation concern. The same business error might warrant different HTTP codes depending on the API endpoint or protocol (REST vs gRPC vs messaging).
    > - **Replaces a meaningful hierarchy with a code parameter** — instead of catching `ProductNotFoundException` or `QuotaExceededException`, every handler catches `ServiceException` and switches on the int code. Type safety and semantic clarity are lost.
    > - **Discourages proper exception modeling** — developers stop creating specific exceptions ("just pass a different code") and the hierarchy degenerates into `ServiceException(message, httpCode)` for everything.
    > - **Fix:** Create semantically named exceptions (`ProductNotFoundException`, `QuotaExceededException`) with `@ResponseStatus` annotations or mapped in `@ControllerAdvice`. The service throws by meaning, the web layer maps to HTTP status.

25. **⚙ Heuristic — Exceptions for expected outcomes instead of Optional/return values** — detect service methods that throw exceptions for predictable, non-fault business outcomes (e.g., "entity not found") where an `Optional` or return value would be cleaner.

    > **⚠ Heuristic check:** This detection is approximate. It flags patterns that *suggest* exception-oriented control flow for expected outcomes, but confirming whether the throw is justified requires **manual review** of the business context. The report provides file paths, line numbers, class names, and method names to facilitate that review.

    a. **Detect `orElseThrow` on repository `findById` calls** — `Grep` on `local_root` for `.orElseThrow(` in service classes (`@Service`). For each match:
       - Read the surrounding context (`-B 3 -A 3`) to determine:
         - Is the method called `findById`, `findByName`, `getById`, or similar lookup method?
         - What exception is thrown? (e.g., `ProductNotFoundException`, `ResourceNotFoundException`)
         - Does the exception represent "not found" semantics (name contains `NotFound`, `Missing`, `NoSuch`)?
       - If yes to all: flag as a heuristic candidate. The service is converting an `Optional.empty()` (a normal return) into an exception.

    b. **Detect try-catch used for flow control** — `Grep` on `local_root` for `catch.*NotFoundException\|catch.*NoSuchElement\|catch.*EntityNotFound` in service classes. For each match, read the catch block:
       - If the catch body creates/returns a default value, redirects, or performs an alternative action (not just rethrow) — this is exception-oriented flow control.
       - Pattern: `try { find(id) } catch (NotFoundException e) { return createDefault(id); }` — the exception replaces a simple `if (exists)` check.

    c. **Detect service methods that always throw on empty** — for each service class, `Grep` for methods containing both a repository call and a `throw new.*NotFoundException` (or similar). If the method's only failure path is "not found" and it throws, it's a candidate.

    d. **Cross-check return types** — via `mcp__imaging__source_file_details` with `nature="inventory"` on service classes, list methods that return a bare entity type (e.g., `Product`) vs `Optional<Product>`. Methods returning bare types that contain `orElseThrow` internally are hiding the optionality from the caller — the caller doesn't know from the signature that "not found" is possible.

    e. For each flagged occurrence, record:
       - **File path** (absolute, for direct navigation)
       - **Line number** (from `Grep -n`)
       - **Class name**
       - **Method name**
       - **Pattern detected** (orElseThrow on findById / try-catch flow control / bare return hiding Optional)
       - **Exception thrown** (the specific exception class)

    > **Why returning Optional is often better than throwing for expected outcomes:**
    > - **"Not found" is not a fault** — it's a normal business outcome. A user searching for a product that doesn't exist is not an error; it's an expected scenario. Exceptions should be reserved for unexpected failures (DB down, timeout, corrupted data).
    > - **The caller loses the choice** — with `Optional`, each caller decides: `.orElse(default)`, `.orElseThrow()`, `.map(...)`, `.ifPresent(...)`. With an exception, the decision is made for them — every caller must catch or propagate.
    > - **Performance** — stack trace construction is expensive. On a high-traffic "not found" path, throwing exceptions wastes CPU on stack frames that will never be read.
    > - **Method signature honesty** — `Optional<Product>` tells the caller that absence is possible. `Product` with an unchecked exception hides this — the caller discovers it at runtime.
    > - **Pollutes business code** — exception-oriented "not found" handling fills service code with try-catch blocks instead of clean if/then or Optional pipelines.
    >
    > **When throwing IS appropriate for "not found":**
    > - The calling context makes absence truly exceptional (e.g., loading the current user's own profile — if it's missing, something is deeply wrong).
    > - The method is a controller-layer orchestrator that has already validated existence and the "not found" indicates a race condition or data inconsistency.
    >
    > **This is a heuristic.** Review each flagged occurrence manually. The report provides exact file locations and method names to make that review efficient.

26. **Exception wrapping without cause chain** — two-phase detection:

    **Phase A — Imaging quality rule (preferred, saves ~80% tokens vs source-only):**
    Query `mcp__imaging__quality_insight_violations` with `nature="iso-5055", id="7652"` — **"Avoid throwing an exception in a catch block without chaining it"** (CWE-778).

    This rule pre-identifies all methods that create a new exception inside a catch block without passing the caught exception as the cause. It returns object IDs, names, and file paths.

    For each violation, use `quality_insight_violations(id="7652", include_locations=True)` to get exact file paths and line numbers.

    **Phase B — Source-based confirmation (optional, for context in the report):**
    For a sample of violations from Phase A (or all, if count is small), read the source at the reported line to extract the exact pattern for the report table:
       - `throw new ServiceException(e.getMessage())` — wraps message, drops cause
       - `throw new ServiceException("context: " + e.getMessage())` — concatenates message, drops cause
       - `throw new ServiceException("context message")` — no cause, no message from original

    > **Why:** Without the cause chain, `exception.getCause()` returns `null` at the handler level. The original stack trace — pointing to the actual line that failed — is gone. Developers debugging production issues see only the wrapper exception and have no idea what triggered it. Always pass the original exception as the cause: `throw new AppException("context", originalException)`.

27. **Returning null instead of throwing on failure** — for each service class (`@Service`):

    a. Use `mcp__imaging__source_file_details` with `nature="inventory"` to list methods. For methods returning an object type (not `void`, not primitive), `Grep` the source file for `return null` within those methods.

    b. For each `return null` found, read the surrounding context to determine if it's on an error/failure path:
       - Inside a catch block: `catch (Exception e) { return null; }` — swallows the exception AND returns null.
       - After a failed lookup: `if (result == null) return null;` — propagates nullity upward.
       - After a conditional check: `if (!isValid) return null;` — failure communicated via null.

    c. Flag methods that return `null` on failure paths when they could instead:
       - Throw a domain exception (`throw new ProductNotFoundException(id)`)
       - Return `Optional.empty()`

    > **Why:** Returning `null` on failure forces every caller to null-check. If any caller forgets, it gets an `NPE` at a distant call site — far from where the failure actually occurred. The null propagates silently through the call chain until something dereferences it. Exceptions or `Optional` make the failure explicit at the point of origin.

28. **Log AND rethrow (duplicate logging)** — for each catch block in service and controller classes:

    a. `Grep` on `local_root` for patterns where a catch block both logs and rethrows:
       - `Grep` for `log(ger)?\.(error|warn).*\n.*throw` (multiline) in service classes
       - Alternatively, read source files of service classes and look for catch blocks containing both a `log.error()`/`log.warn()` call and a `throw` statement.

    b. Flag each occurrence. If `@ControllerAdvice` also logs the exception (check step 7's handler source), the same error generates 2+ log entries.

    > **Why:** Each layer that catches, logs, and rethrows produces a separate log entry for the same error. In production, monitoring dashboards show 2–3x the real error rate. Log aggregation tools (ELK, Splunk) show duplicate entries that waste storage and confuse triage. **Rule:** log OR rethrow, never both. If you rethrow, let the final handler (usually `@ControllerAdvice`) be the single logging point.

29. **Catching `Throwable`** — `Grep` on `local_root` for `catch\s*\(\s*Throwable`:

    a. For each match, read the context to confirm it's catching `Throwable` (not a subclass).

    b. Flag all occurrences. `Throwable` catches `Error` subclasses (`OutOfMemoryError`, `StackOverflowError`, `NoClassDefFoundError`) — JVM-fatal conditions the application cannot recover from.

    c. Exception: catching `Throwable` in a top-level thread runner or `@Async` uncaught handler is acceptable — it's a last-resort safety net to prevent silent thread death. Note this in the report.

    > **Why:** `catch (Throwable t)` catches `Error` subclasses that signal unrecoverable JVM failures. Catching `OutOfMemoryError` and continuing means the JVM is in an undefined state — subsequent allocations may silently corrupt data. The correct response to an `Error` is to let the JVM crash and restart.

30. **Multiple `@ControllerAdvice` without `@Order`** — from step 7, if more than one `@ControllerAdvice`/`@RestControllerAdvice` class was found:

    a. For each advice class, check if it has `@Order(N)` or `@Priority(N)` annotation, or implements `Ordered` interface.

    b. If two or more advice classes exist and any lack an `@Order` annotation, flag — Spring's resolution order is undefined (depends on classpath scanning order, which varies by environment).

    c. Check for overlapping exception handlers: if `AdviceA` and `AdviceB` both have `@ExceptionHandler(SomeException.class)`, the one with higher priority (lower `@Order` number) wins — but without `@Order`, the winner is non-deterministic.

    > **Why:** Adding a new `@ControllerAdvice` class can silently shadow existing handlers. In development it works because classpath order is stable; in production (different JVM, different packaging) the other advice wins and the error response format changes. Always use `@Order` when multiple advice classes coexist.

31. **`@Transactional` + checked exceptions = no rollback** — cross-reference with application exceptions from step 4:

    a. Identify all application exceptions that extend `Exception` (checked) — from the hierarchy built in step 6.

    b. For each `@Transactional` method (query via `mcp__imaging__objects` with `filters="annotation:contains:Transactional"`), check:
       - Does the method throw a checked application exception? (`Grep` the source for `throws <CheckedException>` in the method signature)
       - Does the `@Transactional` annotation include `rollbackFor` that covers this exception? (`Grep` for `rollbackFor` in the annotation)

    c. Flag any `@Transactional` method that throws a checked exception without a matching `rollbackFor` — Spring's default behavior commits on checked exceptions, causing **partial data persistence** on failure.

    > **Why:** Spring `@Transactional` only rolls back on unchecked exceptions (`RuntimeException` and subclasses) by default. If a service method is `@Transactional` and throws a checked `ServiceException extends Exception`, the transaction **commits** — leaving the database in a partially modified state. This is one of the most dangerous Spring gotchas. Fix: add `@Transactional(rollbackFor = ServiceException.class)` or, better, make the exception unchecked.

32. **Exception message as business logic** — `Grep` on `local_root` for patterns where exception messages are parsed to make decisions:

    a. Search for: `getMessage().contains(`, `getMessage().equals(`, `getMessage().startsWith(`, `getMessage().matches(`
    b. Search for: `e.getMessage().indexOf(` or string comparison on exception messages.
    c. Also search for: `catch.*Exception.*\n.*if.*getMessage` (catch block followed by if-on-message).

    d. For each match, read the context. Flag if the code branches on the exception message content to determine error type or action.

    > **Why:** Exception messages are implementation details — they change across library versions, locales, or refactors. Code that parses `e.getMessage().contains("duplicate key")` to detect constraint violations will break silently when the DB driver changes its message format. Use exception types or error codes instead: catch `DuplicateKeyException` directly, or check `SQLException.getSQLState()`.

33. **Error response missing correlation/trace ID** — for each error response DTO (from step 14) and each `@ExceptionHandler` method (from steps 7–8):

    a. Check if the error DTO has a field for correlation/request/trace ID (common names: `traceId`, `requestId`, `correlationId`, `reference`, `instance`).
    b. Check if `@ExceptionHandler` methods inject or generate a request ID (e.g., from MDC, `HttpServletRequest` header, `UUID.randomUUID()`).
    c. Flag if neither the DTO nor the handler includes a correlation ID.

    > **Why:** When a client receives `{"status": 500, "message": "Internal error"}`, they have no reference to give support. The server logs contain the full stack trace but no way to match it to the client's specific request. A correlation ID (e.g., `"reference": "req-abc123"`) bridges the gap — the client reports the reference, support greps the logs. This is especially critical in microservice architectures where a single client request fans out to multiple services.

34. **Async error handling gaps** — detect `@Async` methods and `CompletableFuture` usage without error handling:

    a. `mcp__imaging__objects` with `filters="annotation:contains:Async,type:contains:method"` — list all `@Async` methods.
    b. For each `@Async` method, check:
       - Return type: if `void`, exceptions are silently lost unless an `AsyncUncaughtExceptionHandler` is configured.
       - Return type: if `CompletableFuture` or `Future`, the exception is captured in the future — but the caller must call `.get()` or `.exceptionally()` to observe it.
    c. Check if `AsyncUncaughtExceptionHandler` is configured: `Grep` on `local_root` for `AsyncUncaughtExceptionHandler` or `AsyncConfigurer`.
    d. `Grep` for `CompletableFuture` in service classes. For each usage, check if `.exceptionally(`, `.handle(`, or `.whenComplete(` is called — if not, exceptions from the async chain are silently dropped.

    > **Why:** `@Async void` methods that throw exceptions silently discard them — no log entry, no error response, no notification. The caller never knows the operation failed. For `CompletableFuture`, if no terminal error handler (`.exceptionally()`, `.handle()`) is attached, the exception sits uncollected in the future object until GC collects it. In both cases, data may be partially processed with no indication of failure.

35. **Rethrowing as generic exception** — `Grep` on `local_root` for catch blocks that wrap specific exceptions in generic ones:

    a. Search for patterns like:
       - `catch (SpecificException e) { throw new RuntimeException(e)` — wraps domain exception in generic RuntimeException
       - `catch (SpecificException e) { throw new Exception(e)` — wraps in generic checked Exception
       - `catch (SpecificException e) { throw new RuntimeException(e.getMessage())` — even worse: generic + loses cause

    b. For each match, check if the original exception type has a specific `@ExceptionHandler` — if yes, the rethrow as `RuntimeException` means that handler will never fire.

    > **Why:** Wrapping `PaymentDeclinedException` in `RuntimeException` strips the semantic meaning. `@ControllerAdvice` can no longer match it with `@ExceptionHandler(PaymentDeclinedException.class)` — it falls to the generic catch-all handler and returns 500 instead of 402. The specific exception type is how the system communicates *what went wrong*; wrapping in `RuntimeException` says "something went wrong" — useless.

36. **`return` inside `finally` block** — `Grep` on `local_root` for `finally` blocks containing `return`:

    a. Use multiline `Grep` for `finally\s*\{[^}]*return` or search for `return` statements within finally blocks.
    b. Alternatively, `Grep` for `finally` and read context (`-A 10`) to check for `return` statements inside.

    c. Flag every occurrence — a `return` in `finally` silently swallows any exception thrown in the `try` or `catch` block. The exception simply disappears.

    > **Why:** If `try` throws an exception and `finally` contains a `return`, the exception is silently discarded — the method returns the `finally` value as if nothing went wrong. This is one of the most insidious Java bugs because there is no error, no log, no warning — the exception vanishes completely. Even experienced developers miss this. The JLS specifies this behavior (§14.20.2) but it is almost never intentional.

37. **`@ResponseStatus` on exception + `@ExceptionHandler` for same exception (conflicting handling)** — cross-reference:

    a. From step 4, list all application exceptions that have `@ResponseStatus`.
    b. From steps 7–8, list all exceptions covered by `@ExceptionHandler`.
    c. Find the intersection — exceptions that have BOTH `@ResponseStatus` AND an `@ExceptionHandler`.

    d. For each conflict, check if the `@ResponseStatus` code matches the status returned by the `@ExceptionHandler`. If different, flag as contradictory.

    > **Why:** `@ExceptionHandler` takes priority over `@ResponseStatus` (Spring's `ExceptionHandlerExceptionResolver` runs before `ResponseStatusExceptionResolver`). A developer reading the exception class sees `@ResponseStatus(NOT_FOUND)` and assumes 404 — but the actual response might be 400 because the handler overrides it. If someone later removes the handler (thinking it's redundant), the behavior silently changes. This is contradictory metadata that should be resolved: pick one mechanism, remove the other.

38. **Multi-layer catch (same exception caught at multiple layers)** — for each application exception with callers > 0 (from step 5c):

    a. Use `mcp__imaging__object_details` with `focus="inward"` on the exception class to find all throw sites.
    b. For each throw site, trace the call chain upward (throw site → service method → controller method → `@ControllerAdvice`). Check if the exception is caught at more than one layer:
       - Caught in the repository/DAO layer
       - Caught again in the service layer
       - Caught again in the controller or `@ControllerAdvice`

    c. Alternatively, `Grep` on `local_root` for `catch.*<ExceptionName>` and group matches by architectural layer (service, controller, advice). Flag exceptions caught at 2+ layers.

    d. For each multi-layer catch, read the catch blocks to determine what each layer does (log, wrap, rethrow, swallow, return error). Flag if multiple layers log the same exception (duplicate logging, see also check #28).

    > **Why:** Each layer that catches adds wrapping, logging, and potential behavior changes. By the time the exception reaches `@ControllerAdvice`, it may be wrapped 2–3 levels deep, logged 3 times, and the original context is buried. The correct pattern: catch only where you can **add value** (convert technical → domain exception, enrich with context, or recover). Pass through layers that cannot meaningfully handle the exception.

39. **Inconsistent error logging levels** — for each catch block that logs an exception (from steps 28 and 38):

    a. Classify the caught exception by nature:
       - **Business/expected:** exceptions representing expected outcomes — `ProductNotFoundException`, `ValidationException`, `DuplicateKeyException`, `AccessDeniedException`. These are part of normal application flow.
       - **Infrastructure/system:** exceptions representing unexpected failures — `ConnectException`, `TimeoutException`, `DataAccessException` (non-constraint), `IOException`, `OutOfMemoryError`.

    b. Check the logging level used in the catch block:

    | Exception nature | Logged as | Assessment |
    |-----------------|-----------|-----------|
    | Business/expected | `log.error(...)` | Mismatch — inflates error dashboards, triggers false alerts |
    | Business/expected | `log.warn(...)` or `log.debug(...)` | Correct |
    | Infrastructure/system | `log.warn(...)` | Mismatch — under-alarming, outages go unnoticed |
    | Infrastructure/system | `log.error(...)` | Correct |

    c. For `@ExceptionHandler` methods in `@ControllerAdvice`, apply the same classification: handlers for `NotFoundException` (expected, 404) should log at WARN/DEBUG, handlers for `Exception` (catch-all, 500) should log at ERROR.

    > **Why:** Logging `ProductNotFoundException` as ERROR fires alerts for expected 404s — noise. Logging `ConnectionTimeoutException` as WARN means real outages go unnoticed in monitoring. **Rule:** business exceptions → WARN or DEBUG. Infrastructure failures → ERROR.

---

## Report Template

### Overview
| Metric | Count |
|--------|-------|
| Total exception classes found | N |
| Application exceptions (source available) | N |
| Library exceptions (third-party, no source) | N |
| JDK / Java EE exceptions | N |
| Duplicate exception names (same name, different packages) | N |
| Semantically duplicate exceptions | N |
| Unused exceptions (0 callers) | N |
| @ControllerAdvice / @RestControllerAdvice classes | N |
| @ExceptionHandler methods (global) | N |
| @ExceptionHandler methods (controller-local) | N |
| Unhandled application exceptions | N |
| Controller methods with generic catch | N |
| Controller methods returning Object/String | N |
| Error response DTOs | N |

### Part 1 — Custom Exception Inventory

#### Application Exceptions (source available)
| Exception Class | Package | Extends | @ResponseStatus | HTTP Status | Callers | File |
|----------------|---------|---------|----------------|------------|---------|------|
| ProductNotFoundException | com.myapp.catalog.exception | RuntimeException | @ResponseStatus(NOT_FOUND) | 404 | 5 | path/to/File.java |
| ServiceException | com.myapp.common | RuntimeException | — | — | 12 | path/to/File.java |

#### Library Exceptions (third-party, source unavailable)
| Exception Class | Package | Library |
|----------------|---------|---------|
| HttpClientErrorException | org.springframework.web.client | Spring Web |
| FeignException | feign | OpenFeign |

#### JDK / Java EE Exceptions Referenced
| Exception Class | Package | Usage Count |
|----------------|---------|-------------|
| IllegalArgumentException | java.lang | N callers |
| EntityNotFoundException | javax.persistence | N callers |

#### Duplicate Exceptions — Same Name, Different Packages
| Exception Name | Occurrences | Packages | Recommendation |
|---------------|------------|----------|---------------|
| ProductNotFoundException | 2 | com.myapp.catalog.exception, com.myapp.order.exception | Consolidate into a single shared exception |
| ValidationException | 3 | com.myapp.api, com.myapp.service, com.myapp.common | Pick one, delete the rest |

> **Why duplicates matter:** When multiple exceptions share the same simple name, developers import the wrong one (IDE auto-import picks the first match). Catch clauses may silently miss the intended exception because they catch a different class with the same name. This causes bugs that are extremely hard to diagnose.

#### Duplicate Exceptions — Same Semantic Category
| Category | Exceptions | Recommendation |
|----------|-----------|---------------|
| Not Found | ResourceNotFoundException, EntityNotFoundException, ProductNotFoundException, ItemNotFoundException | Consolidate into one `ResourceNotFoundException(String resourceType, Object id)` |
| Validation | InvalidInputException, ValidationException, BadRequestException | Consolidate into one `ValidationException(List<FieldError> errors)` |
| Unauthorized | UnauthorizedException, ForbiddenException, AccessDeniedException | Consolidate into one per HTTP status (401 vs 403) |

#### Unused Exceptions (0 callers — dead code)
| Exception Class | Package | File |
|----------------|---------|------|
| LegacyOrderException | com.myapp.order | path/to/File.java |

#### Exception Hierarchy
```
RuntimeException
 ├─ BaseAppException (custom base)
 │   ├─ ResourceNotFoundException
 │   ├─ ValidationException
 │   └─ ServiceException
 │       └─ PaymentServiceException
 └─ (direct, no custom base)
     ├─ ProductNotFoundException
     └─ OrderException
Exception (checked)
 └─ ImportException
```

| Check | Finding | Assessment |
|-------|---------|-----------|
| Common base exception | Yes: `BaseAppException` / **No: flat hierarchy** | Flat = no single catch for all app exceptions |
| Mixed checked/unchecked | N checked + M unchecked | Inconsistent if both > 0 |
| Hierarchy depth | Max N levels | Over-engineered if > 3 |

### Part 2 — REST API Error Handling

#### Exception Handler Summary
| Metric | Count |
|--------|:-----:|
| @ControllerAdvice / @RestControllerAdvice classes | N |
| @ExceptionHandler methods (global — in advice classes) | N |
| @ExceptionHandler methods (controller-local) | N |
| **Total @ExceptionHandler methods** | **N** |
| Distinct exception types handled | N |
| Application exceptions with @ResponseStatus (auto-mapped) | N |
| Application exceptions covered (handler or @ResponseStatus) | N / M |
| Application exceptions **not** covered (fall through to default) | N |

#### @ControllerAdvice / @RestControllerAdvice Classes
| Class | Annotation | @ExceptionHandler methods | Exceptions handled | Return type | File |
|-------|-----------|--------------------------|-------------------|------------|------|
| GlobalExceptionHandler | @RestControllerAdvice | 5 | ProductNotFoundException, ValidationException, Exception, ... | ResponseEntity<ErrorResponse> | path/to/File.java |

#### @ExceptionHandler Methods (Global)
| Advice Class | Method | Catches | Return Type | HTTP Status | File |
|-------------|--------|---------|-------------|------------|------|
| GlobalExceptionHandler | handleNotFound | ProductNotFoundException | ResponseEntity<ErrorResponse> | 404 | path/to/File.java |
| GlobalExceptionHandler | handleValidation | MethodArgumentNotValidException | ResponseEntity<ErrorResponse> | 400 | path/to/File.java |
| GlobalExceptionHandler | handleGeneric | Exception | ResponseEntity<ErrorResponse> | 500 | path/to/File.java |

#### @ExceptionHandler Methods (Controller-Local)
| Controller | Method | Catches | Return Type | HTTP Status | File |
|-----------|--------|---------|-------------|------------|------|
| OrderController | handleOrderError | OrderException | ResponseEntity<String> | 400 | path/to/File.java |

> **Anti-pattern:** Controller-local `@ExceptionHandler` methods create inconsistent error handling. Some endpoints return the global `ErrorResponse` format, others return a different shape. Clients cannot rely on a single error schema.

#### Unhandled Application Exceptions
| Exception Class | Thrown by (callers) | @ResponseStatus? | Handled by @ExceptionHandler? | Fate |
|----------------|-------------------|-----------------|------------------------------|------|
| ServiceException | 12 call sites | No | No | Falls through to Spring default → 500 + stack trace |
| ImportException | 3 call sites | No | No | Falls through → 500 |

> **Why:** Unhandled exceptions result in Spring's default error response: HTTP 500, `DefaultErrorAttributes` body with `timestamp`, `status`, `error`, `message`, `path`. In dev/debug profiles, this includes the full stack trace — an information disclosure risk.

#### Generic Try-Catch in Controllers
| Controller | Method | Catch Clause | Action | Assessment |
|-----------|--------|-------------|--------|-----------|
| ProductController | createProduct | `catch(Exception e)` | Returns `ResponseEntity.status(500).body("Error")` | Anti-pattern — bypasses @ControllerAdvice |
| OrderController | processOrder | `catch(Exception e)` | Logs + swallows | **Critical** — silent failure |
| PaymentController | charge | `catch(PaymentException e)` | Returns typed error DTO | Acceptable but prefer @ExceptionHandler |

> **Why generic catch is an anti-pattern in controllers:** It bypasses `@ControllerAdvice`, so the error response format differs from globally-handled exceptions. It also catches exceptions that should propagate (e.g., `NullPointerException`, Spring's own exceptions). The correct pattern is to let exceptions propagate and handle them in `@ControllerAdvice`.

### Part 3 — Polymorphic Response Anti-Patterns

#### Handler Methods with Problematic Return Types
| Controller | Method | HTTP | Return Type | Assessment |
|-----------|--------|------|-------------|-----------|
| ProductController | createProduct | POST | `ResponseEntity<Object>` | Anti-pattern — can return ProductDto or error Map |
| OrderController | getOrder | GET | `String` | Anti-pattern — may return JSON string or error message |
| UserController | login | POST | `ResponseEntity<?>` | Warning — wildcard type |
| ReportController | generate | GET | `Map<String, Object>` | Anti-pattern — unstructured |

#### Polymorphic Response Bodies (success vs error type mismatch)
| Controller | Method | Success Body Type | Error Body Type | File |
|-----------|--------|------------------|----------------|------|
| ProductController | createProduct | ProductDto | Map<String, String> | path/to/File.java |
| OrderController | placeOrder | OrderDto | String | path/to/File.java |

> **Why this matters:** When the same endpoint returns different types for success and error, the API contract is broken. A client calling `POST /products` might receive a `ProductDto` JSON on 200 or a `{"error": "..."}` map on 400 — but the response type is `Object`, so the client must do `instanceof` checks. OpenAPI/Swagger cannot document both schemas for the same status code. The fix: always return `ResponseEntity<ProductDto>` and let `@ControllerAdvice` return `ResponseEntity<ErrorResponse>` for errors — the HTTP status code (2xx vs 4xx/5xx) tells the client which schema to deserialize.

#### Manual JSON String Construction
| Controller | Method | Pattern | File |
|-----------|--------|---------|------|
| HealthController | check | `return "{\"status\":\"UP\"}"` | path/to/File.java |
| AuthController | login | `objectMapper.writeValueAsString(tokenMap)` | path/to/File.java |

> **Why:** Manual JSON string construction bypasses Spring's `HttpMessageConverter` pipeline — no content negotiation, no consistent serialization settings (date format, null handling), no interceptor hooks. It is also fragile: missing escaping causes malformed JSON. Return a typed DTO and let Jackson serialize it.

### Part 4 — Error Response Consistency

#### Error Response DTOs Found
| Class | Fields | Used by | RFC 7807 compliant? | File |
|-------|--------|---------|-------------------|------|
| ErrorResponse | timestamp, status, message, path | GlobalExceptionHandler | Partial (missing `type`, `title`) | path/to/File.java |
| ApiError | code, message, details | OrderController (local) | No | path/to/File.java |

| Check | Finding | Assessment |
|-------|---------|-----------|
| Single error DTO across all handlers | Yes / **No (N different DTOs)** | Inconsistent if > 1 |
| RFC 7807 ProblemDetail usage | Yes / No | Recommended for Spring 6+ |
| @ControllerAdvice return type consistency | All return ErrorResponse / **Mixed types** | Inconsistent if mixed |

#### RFC 7807 Compliance Gap
| Error DTO | `type` (URI) | `title` (String) | `status` (int) | `detail` (String) | `instance` (URI) | Gap count |
|-----------|:---:|:---:|:---:|:---:|:---:|:---------:|
| ErrorResponse | ✗ | ✗ | ✓ | ✓ (as `message`) | ✗ | 3/5 |
| ProblemDetail (Spring) | ✓ | ✓ | ✓ | ✓ | ✓ | 0/5 |

> **RFC 7807 (`application/problem+json`)** defines a standard error shape: `type` (error URI), `title` (human summary), `status` (HTTP code as int), `detail` (occurrence-specific text), `instance` (correlation URI). Custom DTOs missing these fields force clients to handle non-standard shapes per API.

#### Error Code Type Mismatch
| Class | Field | Declared Type | Expected Type | Assessment |
|-------|-------|--------------|---------------|-----------|
| ErrorEntity | errorCode | String | int | Mismatch — HTTP status stored as String |
| GenericRuntimeException | errorCode | String | int | Mismatch — must parse to compare with HTTP status |

> **Why:** HTTP status codes are integers (RFC 7231). Storing them as `String` in error DTOs or exception fields invites parsing errors and allows non-numeric values (e.g., `"INTERNAL_ERROR"`) that clients cannot map to HTTP semantics.

#### Internal Detail Leakage in Error Handlers
| Advice/Controller | Method | Leaking Pattern | Risk |
|------------------|--------|----------------|------|
| GlobalExceptionHandler | handleGeneric | `exception.getMessage()` as response body | Leaks internal class names, SQL, file paths |
| GlobalExceptionHandler | handleGeneric | `exception.getCause().getMessage()` | Leaks root cause (JDBC errors, ORM details) |

> **Why:** Exposing `exception.getMessage()` directly to API clients is an information disclosure vulnerability (OWASP A01:2021). Return a generic user-facing message and log the full exception server-side.

#### NPE Risk in Error Handlers
| Advice/Controller | Method | Unsafe Pattern | Consequence |
|------------------|--------|---------------|-------------|
| GlobalExceptionHandler | handleGeneric | `Objects.requireNonNull(exception.getCause())` | NPE if no cause → 500 from the handler itself |
| GlobalExceptionHandler | handleGeneric | `exception.getCause().getMessage()` without null guard | NPE → handler crashes |

> **Why:** `Throwable.getCause()` returns `null` when no cause was set. An NPE in an error handler turns a handled exception into an unhandled `NullPointerException`, returning Spring's default error response with potential stack trace leakage.

#### Parallel Exception Hierarchies
| Hierarchy | Root Class | Type | Fields | Subclasses |
|-----------|-----------|------|--------|------------|
| 1 | GenericRuntimeException | Unchecked | errorCode (String), errorMessage (String) | ServiceRuntimeException, ... |
| 2 | ServiceException | Checked | exceptionType (int), messageCode (String) | ... |

| Check | Finding | Assessment |
|-------|---------|-----------|
| Number of parallel hierarchies | N | Problematic if > 1 |
| Field structure consistency | Matching / **Divergent** | Divergent = catch handlers must branch on exception type |
| Checked/Unchecked mix | Both present / Single type | Mixed = inconsistent caller contracts |

> **Why:** Parallel hierarchies with different field structures fracture the error model. Developers must remember which to use, handlers cover one but not the other, and response construction logic must branch on exception type.

#### @ControllerAdvice Scope Analysis
| Advice Class | Scope Attribute | Scope Value | Controllers Covered | Controllers Outside Scope |
|-------------|----------------|-------------|:-------------------:|:-------------------------:|
| GlobalExceptionHandler | basePackages | com.myapp.api | 8 | 2 (com.myapp.admin) |

> **Why:** A scoped `@ControllerAdvice` silently excludes controllers in other packages. New modules added later are unprotected unless someone updates the advice scope.

#### Service Methods Returning Error DTOs (Layer Violation)
| Service Class | Method | Return Type | Assessment | File |
|--------------|--------|-------------|-----------|------|
| ProductService | findProduct | ErrorEntity | Anti-pattern — service knows about error presentation | path/to/File.java |
| OrderService | placeOrder | ResponseEntity<Order> | Anti-pattern — HTTP concern in service layer | path/to/File.java |
| PaymentService | charge | Object | Anti-pattern — polymorphic return to carry errors | path/to/File.java |

> **Why:** The service layer should be HTTP-agnostic. Returning error DTOs or `ResponseEntity` from services:
> - **Couples business logic to HTTP presentation** — the service decides how errors look to API clients.
> - **Bypasses @ControllerAdvice** — errors as return values cannot be intercepted by exception handlers.
> - **Forces callers to inspect return values** — `if (result instanceof ErrorResponse)` instead of clean exception flow.
> - **Breaks reusability** — message listeners, schedulers, and other services receive HTTP-shaped error objects that are meaningless outside the web layer.
>
> **Fix:** Replace error DTO returns with domain exceptions (`throw new ProductNotFoundException(id)`). Let `@ControllerAdvice` convert exceptions to the error response DTO.

#### Exceptions Carrying Runtime HTTP Status Codes
| Exception Class | Field/Param | Type | How status is decided | Assessment |
|----------------|-------------|------|----------------------|-----------|
| ServiceException | errorCode (constructor param) | String | Service passes `"404"`, `"500"` at throw-time | Anti-pattern — service decides HTTP status |
| GenericRuntimeException | errorCode (constructor param) | String | Service passes `"500"` default, overridden per throw site | Anti-pattern — generic exception replaces hierarchy |
| ProductNotFoundException | — | — | `@ResponseStatus(NOT_FOUND)` on class | Acceptable — compile-time, declarative |

| Pattern | Count | Assessment |
|---------|:-----:|-----------|
| `@ResponseStatus` on exception class (compile-time) | N | ✓ Acceptable |
| `extends ResponseStatusException` (domain-specific subclass) | N | ✓ Acceptable |
| Runtime HTTP code via constructor parameter | N | ⚠ Anti-pattern |
| Generic exception used for all HTTP statuses | N | ⚠ Anti-pattern |

> **Why exceptions should NOT carry runtime HTTP codes:**
> - **Service makes an HTTP decision** — choosing 404 vs 409 vs 429 is a presentation concern. The same business error might warrant different HTTP codes depending on the protocol (REST vs gRPC vs messaging).
> - **Replaces a meaningful hierarchy** — instead of catching `ProductNotFoundException`, handlers catch `ServiceException` and switch on the int code. Type safety and semantic clarity are lost.
> - **Discourages proper exception modeling** — developers stop creating specific exceptions ("just pass a different code") and the hierarchy degenerates into `ServiceException(message, httpCode)`.
>
> **Acceptable:** `@ResponseStatus(NOT_FOUND)` on the exception class or `extends ResponseStatusException` for domain-specific subclasses — the status is part of the exception's identity, not a runtime decision by the service.
>
> **Fix:** Create semantically named exceptions with `@ResponseStatus` or mapped in `@ControllerAdvice`. The service throws by meaning, the web layer maps to HTTP status.

#### ⚙ Heuristic — Exceptions for Expected Outcomes (Manual Review Required)

> **⚠ This section flags patterns that suggest exception-oriented control flow for expected business outcomes. Each occurrence requires manual review to confirm — the automated detection cannot determine business intent. File paths, line numbers, and method names are provided for efficient review.**

| # | File | Line | Class | Method | Pattern | Exception Thrown |
|---|------|:----:|-------|--------|---------|-----------------|
| 1 | `src/main/.../ProductServiceImpl.java` | 42 | ProductServiceImpl | getProduct | `orElseThrow` on `findById` | ProductNotFoundException |
| 2 | `src/main/.../OrderServiceImpl.java` | 87 | OrderServiceImpl | findOrder | `orElseThrow` on `findById` | ResourceNotFoundException |
| 3 | `src/main/.../UserServiceImpl.java` | 23 | UserServiceImpl | loadUser | try-catch flow control | UserNotFoundException → creates default |
| 4 | `src/main/.../CatalogServiceImpl.java` | 115 | CatalogServiceImpl | getCatalog | bare return type hides Optional | CatalogNotFoundException |

**Summary:**
| Pattern | Occurrences | Assessment |
|---------|:-----------:|-----------|
| `orElseThrow` on repository lookup (converts Optional.empty → exception) | N | Review — is "not found" truly exceptional here? |
| Try-catch for flow control (catch → alternative action) | N | Likely anti-pattern — replace with `if`/`Optional` |
| Bare return type hiding Optional from caller | N | Review — caller unaware that absence is possible |
| **Total candidates for manual review** | **N** | |

> **Why Optional is often better than throwing for expected outcomes:**
> - "Not found" is a normal business outcome, not a fault — exceptions should be reserved for unexpected failures.
> - The caller loses the choice: with `Optional`, each decides (`.orElse()`, `.map()`, `.orElseThrow()`). With an exception, the decision was already made.
> - Stack trace construction is expensive on high-traffic "not found" paths.
> - `Optional<Product>` in the signature is honest — the caller knows absence is possible. A bare `Product` with a hidden unchecked exception is a surprise.
>
> **When throwing IS appropriate:** if the calling context makes absence truly exceptional (e.g., loading the authenticated user's own profile — if missing, something is deeply wrong).

### Part 5 — Exception Hygiene

#### Exception Wrapping Without Cause Chain
| Class | Method | Line | Pattern | File |
|-------|--------|:----:|---------|------|
| OrderServiceImpl | placeOrder | 87 | `throw new ServiceException(e.getMessage())` — cause dropped | path/to/File.java |
| PaymentService | charge | 54 | `throw new AppException("Payment failed: " + e.getMessage())` — cause dropped | path/to/File.java |

**Total: N occurrence(s)**

> **Why:** Without the cause chain (`throw new XException("msg", e)`), `exception.getCause()` returns `null`. The original stack trace — pointing to the actual line that failed — is permanently lost. Always pass the original exception as the second constructor argument.

#### Returning Null Instead of Throwing on Failure
| Class | Method | Line | Context | File |
|-------|--------|:----:|---------|------|
| ProductServiceImpl | findProduct | 42 | `catch (DataAccessException e) { return null; }` | path/to/File.java |
| UserService | getUser | 28 | `if (result == null) return null;` | path/to/File.java |

**Total: N occurrence(s)**

> **Why:** Returning `null` on failure forces every caller to null-check. If any caller forgets, it gets an NPE far from where the failure occurred. The null propagates silently through the call chain. Use `Optional.empty()` or throw a domain exception to make failure explicit at the point of origin.

#### Log AND Rethrow (Duplicate Logging)
| Class | Method | Line | Logs as | Then | File |
|-------|--------|:----:|--------|------|------|
| OrderServiceImpl | processOrder | 95 | `log.error("Order failed", e)` | `throw e` | path/to/File.java |
| PaymentService | charge | 62 | `log.warn("Payment error", e)` | `throw new PaymentException(e)` | path/to/File.java |

**Total: N occurrence(s)**

> **Why:** Each layer that logs and rethrows produces a separate log entry for the same error. Monitoring dashboards show 2–3x the real error rate. **Rule:** log OR rethrow, never both. Let the final handler (`@ControllerAdvice`) be the single logging point.

#### Catching Throwable
| Class | Method | Line | Action in catch block | File |
|-------|--------|:----:|-----------------------|------|
| BatchProcessor | run | 110 | `catch (Throwable t) { log.error(...) }` | path/to/File.java |

**Total: N occurrence(s)**

> **Why:** `catch (Throwable)` catches `OutOfMemoryError`, `StackOverflowError`, `NoClassDefFoundError` — JVM-fatal conditions the application cannot recover from. Catching and continuing means the JVM is in an undefined state. **Exception:** acceptable in top-level thread runners or `@Async` uncaught handlers as a last-resort safety net.

#### Multiple @ControllerAdvice Without @Order
| Advice Class | Has @Order? | Order value | Exceptions handled | File |
|-------------|:-----------:|:-----------:|:------------------:|------|
| GlobalExceptionHandler | No | — | Exception, NotFoundException, ... | path/to/File.java |
| ValidationAdvice | No | — | MethodArgumentNotValidException, ... | path/to/File.java |

| Check | Finding | Assessment |
|-------|---------|-----------|
| Multiple @ControllerAdvice classes | N classes | Needs @Order if > 1 |
| Overlapping exception handlers across advice classes | N overlap(s) | Non-deterministic winner without @Order |

> **Why:** Without `@Order`, Spring picks a handler non-deterministically when two advice classes handle the same exception type. Adding a new advice class can silently shadow existing handlers. The behavior may differ between dev and prod (different classpath scanning order).

#### @Transactional + Checked Exceptions = No Rollback
| Class | Method | @Transactional | Throws (checked) | rollbackFor includes it? | File |
|-------|--------|:--------------:|-------------------|:------------------------:|------|
| OrderService | placeOrder | Yes | ServiceException | No — **will commit on failure** | path/to/File.java |
| ImportService | importData | Yes | ImportException | No — **will commit on failure** | path/to/File.java |

**Total: N method(s) at risk of partial commit**

> **Why:** Spring `@Transactional` only rolls back on **unchecked** exceptions by default. If a method throws a checked exception, the transaction **commits** — leaving the database in a partially modified state. Fix: `@Transactional(rollbackFor = ServiceException.class)` or make the exception unchecked.

#### Exception Message as Business Logic
| Class | Method | Line | Pattern | File |
|-------|--------|:----:|---------|------|
| OrderService | processPayment | 73 | `if (e.getMessage().contains("duplicate key"))` | path/to/File.java |
| UserService | register | 45 | `if (e.getMessage().startsWith("Constraint"))` | path/to/File.java |

**Total: N occurrence(s)**

> **Why:** Exception messages are implementation details — they change across library versions, locales, or refactors. Code that parses messages to detect error types will break silently. Use exception types or error codes instead: catch `DuplicateKeyException` directly, or check `SQLException.getSQLState()`.

#### Error Response Missing Correlation/Trace ID
| Error DTO | Has correlation ID field? | Field name | Type |
|-----------|:------------------------:|------------|------|
| ErrorResponse | No | — | — |
| ProblemDetail | Yes (inherited) | `instance` | URI |

| Check | Finding | Assessment |
|-------|---------|-----------|
| Error DTO includes correlation/trace ID | Yes / **No** | Client cannot correlate error with server logs |
| @ExceptionHandler injects request ID | Yes / **No** | No request tracing in error responses |

> **Why:** Without a correlation ID, clients receive `"Internal error"` with no reference for support. Server logs have the stack trace but no way to match it to the client's request. A correlation ID (`"reference": "req-abc123"`) bridges the gap.

#### Async Error Handling Gaps
| Class | Method | Annotation | Return Type | Error handling | File |
|-------|--------|-----------|-------------|---------------|------|
| NotificationService | sendEmail | @Async | void | None — exceptions silently lost | path/to/File.java |
| ReportService | generate | @Async | CompletableFuture | No `.exceptionally()` by any caller | path/to/File.java |

| Check | Finding | Assessment |
|-------|---------|-----------|
| `@Async void` methods (exceptions silently lost) | N method(s) | ⚠ High — silent failure |
| `AsyncUncaughtExceptionHandler` configured | Yes / **No** | Required if any @Async void exists |
| `CompletableFuture` without error handler | N chain(s) | ⚠ Medium — uncollected exceptions |

> **Why:** `@Async void` methods that throw silently discard the exception — no log, no error response. `CompletableFuture` without `.exceptionally()` leaves exceptions uncollected. Data may be partially processed with no indication of failure.

#### Rethrowing as Generic Exception
| Class | Method | Line | Catches | Rethrows as | File |
|-------|--------|:----:|---------|-------------|------|
| ProductService | save | 55 | `PaymentDeclinedException` | `new RuntimeException(e)` | path/to/File.java |
| OrderService | validate | 78 | `ValidationException` | `new RuntimeException(e.getMessage())` — cause also lost | path/to/File.java |

**Total: N occurrence(s)**

> **Why:** Wrapping `PaymentDeclinedException` in `RuntimeException` strips semantic meaning. `@ExceptionHandler(PaymentDeclinedException.class)` will never fire — the exception falls to the generic catch-all handler and returns 500 instead of 402. The specific type is how the system communicates *what went wrong*.

#### `return` Inside `finally` Block
| Class | Method | Line | File |
|-------|--------|:----:|------|
| CacheService | getFromCache | 92 | path/to/File.java |

**Total: N occurrence(s)**

> **Why:** If `try` throws an exception and `finally` contains a `return`, the exception is **silently discarded** — the method returns the `finally` value as if nothing happened. No error, no log, no warning. This is specified by the JLS (§14.20.2) but is almost never intentional. One of the most insidious Java bugs.

#### @ResponseStatus + @ExceptionHandler Conflict
| Exception Class | @ResponseStatus | @ExceptionHandler returns | Actual HTTP status | Contradiction? | File |
|----------------|----------------|--------------------------|:------------------:|:--------------:|------|
| ProductNotFoundException | `@ResponseStatus(NOT_FOUND)` → 404 | `ResponseEntity(BAD_REQUEST)` → 400 | 400 | Yes — @ExceptionHandler wins | path/to/File.java |

**Total: N conflict(s)**

> **Why:** `@ExceptionHandler` takes priority over `@ResponseStatus`. A developer reading the exception class sees 404 but the actual response is 400. If someone later removes the handler, the behavior silently changes. Pick one mechanism, remove the other.

#### Multi-Layer Catch (Same Exception Caught at Multiple Layers)
| Exception | Caught in layer 1 | Action | Caught in layer 2 | Action | Caught in layer 3 | Action |
|-----------|-------------------|--------|--------------------|--------|--------------------|--------|
| DataAccessException | Repository (log + wrap) | → ServiceException | Service (log + wrap) | → AppException | @ControllerAdvice (log + respond) | 3 log entries |

**Total: N exception(s) caught at 2+ layers**

> **Why:** Each layer that catches adds wrapping and logging. By the time the exception reaches `@ControllerAdvice`, it's wrapped 2–3 levels deep and logged multiple times. **Rule:** catch only where you can add value (convert technical → domain, enrich with context, or recover). Pass through layers that can't meaningfully handle the exception.

#### Inconsistent Error Logging Levels
| Class | Method | Exception caught | Logged as | Expected level | File |
|-------|--------|-----------------|-----------|---------------|------|
| OrderService | findOrder | ProductNotFoundException (business, expected) | ERROR | WARN or DEBUG | path/to/File.java |
| PaymentService | charge | ConnectionTimeoutException (system failure) | WARN | ERROR | path/to/File.java |

| Pattern | Count | Assessment |
|---------|:-----:|-----------|
| Business/expected exceptions logged as ERROR | N | Noise — inflates error dashboards |
| System/infrastructure failures logged as WARN | N | Under-alarming — outages go unnoticed |

> **Why:** Logging `ProductNotFoundException` as ERROR fires alerts for expected 404s. Logging `ConnectionTimeoutException` as WARN means real outages go unnoticed in monitoring dashboards. **Rule:** business exceptions (not found, validation, conflict) → WARN or DEBUG. Infrastructure failures (timeout, connection refused, OOM) → ERROR.

## Problem Summary

| # | Check | Issues found | Severity |
|---|-------|-------------|----------|
| 1 | Duplicate exception names (same name, different packages) | N pair(s) | Medium — wrong import risk |
| 2 | Semantically duplicate exceptions | N group(s) | Medium — maintenance burden |
| 3 | Unused exceptions (dead code) | N class(es) | Low — dead code |
| 4 | Flat exception hierarchy (no common base) | Yes / No | Medium — no unified catch |
| 5 | Mixed checked / unchecked exceptions | N checked + M unchecked | Medium — inconsistent contract |
| 6 | Unhandled application exceptions (no @ExceptionHandler, no @ResponseStatus) | N exception(s) | High — 500 + stack trace leak |
| 7 | Controller-local @ExceptionHandler (inconsistent error handling) | N method(s) | Medium — mixed error formats |
| 8 | Generic catch(Exception) in controllers | N method(s) | High — bypasses @ControllerAdvice |
| 9 | Swallowed exceptions (catch with no throw/response) | N method(s) | Critical — silent failure |
| 10 | Polymorphic return types (Object / String / Map / ?) | N method(s) | High — broken API contract |
| 11 | Polymorphic response bodies (success/error type mismatch) | N method(s) | High — undocumentable API |
| 12 | Manual JSON string construction | N method(s) | Medium — fragile, bypasses pipeline |
| 13 | Multiple error response DTOs | N DTO(s) | Medium — inconsistent error shape |
| 14 | No @ControllerAdvice present | Yes / No | High if controllers exist |
| 15 | Missing @ExceptionHandler for common exceptions | MethodArgumentNotValidException, ConstraintViolationException, etc. | Medium — default error leaks info |
| 16 | RFC 7807 compliance gap | N/5 standard fields missing per error DTO | Medium — non-standard error shape |
| 17 | Error code type mismatch (String instead of int) | N field(s) | Medium — parsing errors, type confusion |
| 18 | Internal detail leakage in error messages | N handler(s) expose exception.getMessage() | High — information disclosure (OWASP A01) |
| 19 | NPE risk in error handlers (getCause() without null guard) | N handler(s) | Critical — handler crash → unhandled 500 |
| 20 | Parallel exception hierarchies with divergent fields | N hierarchy pair(s) | Medium — fractured error model |
| 21 | Missing @ExceptionHandler for thrown application exceptions | N exception(s) thrown but unhandled | High — 500 + stack trace leak |
| 22 | @ControllerAdvice scope excludes controllers | N controller(s) outside scope | High — silently unprotected endpoints |
| 23 | Service methods returning error DTOs (layer violation) | N method(s) | High — service coupled to HTTP/presentation |
| 24 | Exceptions carrying runtime HTTP status codes (constructor param) | N exception(s) | Medium — service decides HTTP status, kills hierarchy |
| 25 | ⚙ Heuristic: exceptions for expected outcomes instead of Optional | N occurrence(s) — manual review | Low — design smell, not a bug |
| 26 | Exception wrapping without cause chain | N occurrence(s) | High — original stack trace permanently lost |
| 27 | Returning null instead of throwing on failure | N method(s) | Medium — silent failure, NPE at distant call site |
| 28 | Log AND rethrow (duplicate logging) | N occurrence(s) | Medium — 2–3x error count in monitoring |
| 29 | Catching Throwable | N occurrence(s) | High — catches JVM-fatal errors (OOM, StackOverflow) |
| 30 | Multiple @ControllerAdvice without @Order | N advice class(es) without @Order | Medium — non-deterministic handler priority |
| 31 | @Transactional + checked exception = no rollback | N method(s) | Critical — partial data commit on failure |
| 32 | Exception message as business logic | N occurrence(s) | Medium — fragile, breaks on library/locale changes |
| 33 | Error response missing correlation/trace ID | Yes / No | Medium — client cannot correlate with server logs |
| 34 | Async error handling gaps (@Async void, CompletableFuture) | N method(s) / chain(s) | High — exceptions silently lost |
| 35 | Rethrowing as generic exception (RuntimeException wrapper) | N occurrence(s) | High — strips semantic meaning, bypasses specific handlers |
| 36 | return inside finally block | N occurrence(s) | Critical — silently swallows exceptions |
| 37 | @ResponseStatus + @ExceptionHandler conflict on same exception | N conflict(s) | Medium — contradictory metadata, confusing behavior |
| 38 | Multi-layer catch (same exception caught at 2+ layers) | N exception(s) | Medium — duplicate logging, deep wrapping |
| 39 | Inconsistent error logging levels (business as ERROR, system as WARN) | N mismatch(es) | Low — monitoring noise / missed alerts |

### Priority Fix Order
| Priority | Finding | Recommendation |
|----------|---------|---------------|
| P1 | NPE risk in error handlers (#19) | Add null guards for `getCause()` — handler crash = unhandled 500 |
| P1 | Swallowed exceptions (#9) | Add logging + rethrow, or return a meaningful error response |
| P1 | No @ControllerAdvice (#14) | Create a `@RestControllerAdvice` with handlers for common exceptions |
| P1 | Unhandled application exceptions (#6, #21) | Add `@ResponseStatus` or `@ExceptionHandler` for each thrown exception |
| P2 | Internal detail leakage (#18) | Replace `exception.getMessage()` with generic user-facing message; log full exception server-side |
| P2 | @ControllerAdvice scope excludes controllers (#22) | Remove scope restriction or add advice per package |
| P2 | Generic catch(Exception) in controllers (#8) | Remove try-catch, let @ControllerAdvice handle it |
| P2 | Polymorphic return types (#10) | Change to `ResponseEntity<SpecificDto>`, errors via @ControllerAdvice |
| P2 | Multiple error DTOs (#13) | Consolidate into a single `ErrorResponse` (ideally RFC 7807 `ProblemDetail`) |
| P3 | RFC 7807 compliance gap (#16) | Adopt Spring `ProblemDetail` or add missing standard fields |
| P3 | Error code type mismatch (#17) | Use `int`/`Integer` for HTTP status fields; separate business codes into distinct field |
| P3 | Parallel exception hierarchies (#20) | Consolidate into single hierarchy (prefer unchecked) with consistent fields |
| P3 | Duplicate exceptions (#1, #2) | Consolidate per semantic category, delete redundant classes |
| P3 | Controller-local @ExceptionHandler (#7) | Move to @ControllerAdvice for consistency |
| P2 | Service methods returning error DTOs (#23) | Throw domain exceptions instead; let @ControllerAdvice convert to error response |
| P3 | Exceptions carrying runtime HTTP codes (#24) | Create named exceptions with @ResponseStatus; remove HTTP code constructor params |
| P1 | @Transactional + checked exception = no rollback (#31) | Add `rollbackFor` or make exception unchecked — partial commits corrupt data |
| P1 | return inside finally block (#36) | Remove `return` from `finally` — silently swallows all exceptions |
| P2 | Exception wrapping without cause chain (#26) | Always pass original exception as cause: `new XException("msg", e)` |
| P2 | Catching Throwable (#29) | Change to `catch (Exception e)` — let JVM-fatal errors propagate |
| P2 | Async error handling gaps (#34) | Configure `AsyncUncaughtExceptionHandler`; add `.exceptionally()` to CompletableFuture chains |
| P2 | Rethrowing as generic exception (#35) | Rethrow as domain exception or let original propagate |
| P3 | Returning null instead of throwing (#27) | Return `Optional.empty()` or throw domain exception |
| P3 | Log AND rethrow (#28) | Pick one: log at final handler, or rethrow without logging |
| P3 | Multiple @ControllerAdvice without @Order (#30) | Add `@Order(N)` to each advice class |
| P3 | Exception message as business logic (#32) | Use exception types or error codes instead of parsing messages |
| P3 | Error response missing correlation ID (#33) | Add traceId/requestId field to error DTO; populate from MDC |
| P3 | @ResponseStatus + @ExceptionHandler conflict (#37) | Pick one mechanism per exception, remove the other |
| P3 | Multi-layer catch (#38) | Remove intermediate catches that only log + rethrow |
| P4 | Inconsistent error logging levels (#39) | Business exceptions → WARN/DEBUG; infrastructure failures → ERROR |
| P4 | ⚙ Exceptions for expected outcomes (#25) | Manual review — consider Optional for "not found" lookup patterns |
| P4 | Manual JSON string construction (#12) | Return typed DTOs, let Jackson serialize |
| P4 | Unused exceptions (#3) | Delete dead code |
