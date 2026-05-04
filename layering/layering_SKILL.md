---
name: layering
description: Detect and report architectural layering violations in a Spring application. Checks for Controller to Repository direct calls, Entity exposure at REST endpoints, Entity classes with Jackson annotations, and Service methods returning error DTOs or ResponseEntity.
---

# Layering Violations Review

Detect and report architectural layering violations in a Spring application:

1. **Controller → Repository**: controller classes that call repositories directly, bypassing the service layer.
2. **Entity exposed at REST**: `@Entity` classes referenced directly by controller methods (as parameters, return types, or instantiated objects).
3. **Entity with Jackson annotations**: `@Entity` classes or their fields carrying `@Json*` annotations, mixing persistence and serialization concerns in the same class.
4. **Service → Error DTO / ResponseEntity**: `@Service` classes whose methods return error DTOs or `ResponseEntity`, leaking HTTP/presentation concerns into the business layer.

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
      **Filename:** `<app>-layering-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "layering",
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

3. **Collect baseline inventories** — run all queries below in parallel using `mcp__imaging__objects`, reading **full content**:

   - **Controller classes**: `filters="annotation:contains:Controller,type:contains:class"`
     Post-filter: keep only those whose `annotations` array contains `@RestController` or `@Controller`. Discard `@ControllerAdvice`, `@RestControllerAdvice`, etc.
     Record `id`, `name`, `fullName`.

   - **Repository classes** (two queries, merge and deduplicate by `id`):
     - `filters="annotation:contains:Repository,type:contains:class"`
     - `filters="name:contains:Repository,type:contains:class"`
     Also run: `filters="name:contains:Dao,type:contains:class"`
     Record `id`, `name`, `fullName`.

   - **Entity classes**: `filters="annotation:contains:Entity,type:contains:class"`
     Post-filter: keep only those whose `annotations` array contains an entry that is exactly `@Entity` or starts with `@javax.persistence.Entity` or `@jakarta.persistence.Entity`. Discard `@EntityListeners`, `@EntityScan`, `@EntityGraph`, etc.
     Record `id`, `name`, `fullName`. This confirmed entity ID set is the authoritative reference for all later entity filtering.

   - **Entity classes with class-level Jackson annotations**: `filters="annotation:contains:Json,annotation:contains:Entity,type:contains:class"`
     > ⚠ **Known limitation:** this query uses substring matching. It will return false positives — for example, controllers whose Swagger/OpenAPI annotations contain the word "Entity" in their text, or entities where a *field-level* `@JsonIgnore` is indexed under the class. Do **not** use this query result directly.
     >
     > **Correct approach:**
     > 1. Post-filter by confirmed entity ID set (discard any object whose `id` is not in the entity list above).
     > 2. For the class-level sub-check, inspect each entity's own `annotations` array (the class-level annotations, not its fields). Keep only entries whose annotation text starts with `@Json` (e.g. `@JsonIgnoreProperties`, `@JsonTypeInfo`).
     > 3. Field-level Jackson annotations are detected via the separate field queries below — do not use this class query for field-level detection.
     Record `name`, `fullName`, and which class-level `@Json*` annotations are present.

   - **Fields with Jackson annotations** (run these three in parallel):
     - `filters="annotation:contains:JsonProperty,type:contains:field"`
     - `filters="annotation:contains:JsonIgnore,type:contains:field"`
     - `filters="annotation:contains:JsonSerialize,type:contains:field"` (also catches `@JsonDeserialize`, `@JsonFormat`, `@JsonInclude` via substring)
     For each result, derive the owner class from `fullName` (strip the field name suffix). Post-filter: keep only fields whose owner class is in the confirmed entity class set.

   Paginate all queries if `metadata.has_next: true`.

4. **Enumerate controller methods** — for each controller class ID from step 3, call `mcp__imaging__object_details` with `focus=intra` (in parallel) to retrieve the list of child objects (methods, fields). Build two lookup structures:
   - `controllerMethodIds`: set of all method IDs that belong to any controller class
   - `methodIdToControllerClass`: map of `methodId → controllerClassName`

5. **Check 1 — Controller → Repository direct calls** — for each repository class ID from step 3, call `mcp__imaging__object_details` with `focus=inward` (in parallel). From each result's `incoming_calls`, check whether any caller `id` is in `controllerMethodIds`. For each match:
   - Caller method name = `incoming_calls[].name`
   - Caller controller class = lookup via `methodIdToControllerClass`
   - Called repository = the repository class being queried
   - Link type = `incoming_calls[].linkType` (typically `CALL` for direct method calls or `RELY_ON` for field injection)

   Record each violation as: `ControllerClass.callerMethod → RepositoryClass`.

   > This detects both field injection (controller holds a repository field, `RELY_ON` link) and direct method calls from a controller to a repository (`CALL` link).

6. **Check 2 — @Entity classes exposed at REST endpoints** — for each entity class ID from step 3, call `mcp__imaging__object_details` with `focus=inward` (in parallel). From each result's `incoming_calls`, check whether any caller `id` is in `controllerMethodIds`. For each match, record: `ControllerClass.callerMethod → EntityClass` and the `linkType`.

   > **Link types to count as violations:**
   > - `RELY_ON` — the controller method's signature or field declaration references the entity type (parameter type, return type, field type). This is the most common form: `public ResponseEntity<Order> getOrder(...)` or `@RequestBody Customer customer`.
   > - `CALL` — the controller method directly instantiates or calls a static method on the entity class.
   > Both indicate the entity is directly coupled to the HTTP layer.

   > **Sampling strategy for large apps:** If the entity list is very large (50+ classes), query the first 10–15 most central entities (those referenced in the most places — typically `Customer`, `Order`, `Product`, `MerchantStore`, `User`, `Cart`, etc.) to establish whether the pattern is present. State the sample size in the report. If violations are found in the sample, the pattern almost certainly applies more broadly.

   > A controller method that directly references a `@Entity` class couples the HTTP layer to the persistence model. The entity is likely returned as a response body or accepted as a request body, bypassing a DTO layer. This prevents independent evolution of the API contract and the database schema, and risks accidentally exposing persistence-only fields (foreign key IDs, lazy proxies, audit fields) to API clients.

7. **Check 4 — Service methods returning error DTOs or ResponseEntity (layer violation)** — service classes should be HTTP-agnostic. Returning error DTOs or `ResponseEntity` from a `@Service` class leaks HTTP/presentation concerns into the business layer.

   a. **Identify error response DTOs** — `Grep` on `local_root` for classes named: `ErrorResponse`, `ErrorDto`, `ErrorEntity`, `ApiError`, `ErrorMessage`, `ExceptionResponse`, `ProblemDetail`. Record their simple names.

   b. **Identify service classes** — from `mcp__imaging__objects` with `filters="annotation:contains:Service,type:contains:class"`. Post-filter for exact `@Service`. For each service class, get its method signatures via `mcp__imaging__source_file_details` with `nature="inventory"`, or read the source file directly.

   c. **Flag service methods** whose return type is:
      - An error DTO class (from step 7a)
      - `ResponseEntity` or `ResponseEntity<...>`
      - `Object` used polymorphically (returns either a success DTO or error DTO — confirm by reading the source for branching return paths)

   d. Record each violation as: `ServiceClass.method → returnType`.

   > **Why:** When a service returns an error DTO or `ResponseEntity`, it becomes aware of how errors are presented to HTTP clients. This couples business logic to the transport layer, bypasses `@ControllerAdvice` (errors as return values can't be intercepted), forces callers to inspect return values, and breaks reusability (message listeners and schedulers receive HTTP-shaped errors). The correct pattern: throw domain exceptions, let `@ControllerAdvice` convert them to error responses.

8. **Produce the report** in this structure:

```
## Layering Violations Report

### Check 1 — Controller → Repository Direct Calls
[If none:] No controllers reference repositories directly — ✓

[If found:]
| Controller | Method | Repository | Link type |
|------------|--------|------------|-----------|
| UserController | createUser | UserRepository | CALL |

**Total: N violation(s) across N controller(s)**

> ⚠ **Anti-pattern:** Controllers should delegate all persistence access to the service layer. A direct
> controller → repository dependency means business logic (validation, orchestration, transaction
> boundaries) will either be duplicated, skipped, or placed in the controller — where it cannot be
> reused by other services and is harder to test.
>
> **Fix:** Introduce or use an existing service class as the intermediary. The controller calls the
> service; the service owns the transaction and calls the repository.

---

### Check 2 — @Entity Classes Exposed at REST Endpoints
[If none:] No @Entity classes referenced by controller methods — ✓

[If found:]
| Controller | Method | Entity Class | Link type |
|------------|--------|-------------|-----------|
| OrderController | createOrder | OrderEntity | RELY_ON |

**Total: N violation(s) across N controller(s)**
[If sample-based: note "based on a sample of N/T entities — pattern likely applies more broadly"]

> ⚠ **Anti-pattern:** Exposing JPA entities directly as request/response bodies tightly couples the
> API contract to the database schema. Consequences:
> - Any schema change (added column, renamed field) becomes a breaking API change.
> - Persistence-only fields (e.g. `version`, `createdAt`, foreign key IDs, Hibernate proxy types) leak
>   into the API surface.
> - Bidirectional relationships trigger infinite recursion during JSON serialization unless suppressed
>   with `@JsonIgnore` — which is itself an anti-pattern (see Check 3).
> - `@RequestBody` mapping into an entity and calling `save()` directly risks mass-assignment attacks.
>
> **Fix:** Introduce DTOs or record classes for the API layer. Map between entities and DTOs in the
> service layer (manually or via MapStruct / ModelMapper).

---

### Check 3 — @Entity Classes with Jackson Annotations
[If none:] No @Entity classes carry Jackson annotations — ✓

[If found:]

#### Class-level Jackson annotations
[If none:] None detected — ✓

[If found:]
| Entity Class | Jackson annotations present |
|-------------|----------------------------|
| UserEntity | @JsonIgnoreProperties({"hibernateLazyInitializer"}) |

#### Field-level Jackson annotations
[If none:] None detected — ✓

[If found:]
| Entity Class | Field | Annotation |
|-------------|-------|------------|
| UserEntity | password | @JsonIgnore |
| OrderEntity | items | @JsonSerialize |

**Total: N entity class(es) carrying Jackson annotations**

> ⚠ **Anti-pattern:** Jackson annotations on `@Entity` classes signal that the entity is being used
> directly as a serialization target — which is the root cause flagged in Check 2. The `@Json*`
> annotations are typically added as workarounds for the serialization problems that arise from
> direct entity exposure (infinite recursion, lazy proxies, unwanted fields). Treating the symptom
> (add `@JsonIgnore`) rather than the cause (use a DTO) leads to an ever-growing list of suppression
> annotations and brittle serialization behaviour.
>
> **Fix:** Move all serialization annotations to DTO/record classes. Entity classes should carry only
> JPA annotations (`@Column`, `@OneToMany`, etc.).

---

### Check 4 — Service Methods Returning Error DTOs / ResponseEntity
[If none:] No service methods return error DTOs or ResponseEntity — ✓

[If found:]
| Service Class | Method | Return Type | Assessment |
|--------------|--------|-------------|-----------|
| ProductService | findProduct | ErrorEntity | Anti-pattern — service knows about error presentation |
| OrderService | placeOrder | ResponseEntity<Order> | Anti-pattern — HTTP concern in service layer |

**Total: N violation(s) across N service class(es)**

> ⚠ **Anti-pattern:** Service classes should be HTTP-agnostic. Returning error DTOs or `ResponseEntity`
> couples business logic to the transport layer, bypasses `@ControllerAdvice`, forces callers to inspect
> return values (`instanceof` checks), and breaks reusability (schedulers and message listeners receive
> HTTP-shaped errors).
>
> **Fix:** Throw domain exceptions (e.g., `throw new ProductNotFoundException(id)`) and let
> `@ControllerAdvice` convert them to error response DTOs.

[If running inside `/full-review`:]
> 📎 **Cross-reference:** For a detailed analysis of error DTOs, exception hierarchies, @ControllerAdvice
> coverage, and RFC 7807 compliance, see <a href="#s14">Section 14 — Error Handling</a>.

---

### Summary
| Check | Violations | Assessment |
|-------|-----------|------------|
| Controller → Repository direct calls | N | ⚠ Anti-pattern / ✓ None |
| @Entity classes at REST endpoints | N | ⚠ Anti-pattern / ✓ None |
| @Entity classes with Jackson annotations | N | ⚠ Anti-pattern / ✓ None |
| Service → Error DTO / ResponseEntity | N | ⚠ Anti-pattern / ✓ None |
```

Always report all three checks. A check with no violations must say "— ✓" rather than be omitted.

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | Controller → Repository (direct calls) | N violation(s) | ⚠ Anti-pattern / ✓ None |
| 2 | `@Entity` classes exposed at REST endpoints | N violation(s) | ⚠ Anti-pattern / ✓ None |
| 3 | `@Entity` classes with Jackson annotations | N class(es) | ⚠ Anti-pattern / ✓ None |
| 4 | Service → Error DTO / ResponseEntity | N method(s) | ⚠ Anti-pattern / ✓ None |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | Entities exposed to the REST layer | N entity/entities referenced from controllers | Introduce DTOs/records; map in the service layer |
| 2 | Service returning error DTOs / ResponseEntity | N method(s) returning HTTP-layer types | Throw domain exceptions; let @ControllerAdvice handle error responses |
| 3 | Controller → Repository (bypass service) | N violation(s) | Delegate to service; remove the direct dependency |
| 4 | Jackson annotations on JPA entities | N entity/entities | Move annotations to DTOs |

> Replace N values above with actual counts from the preceding checks. Remove rows where count is 0 and severity is ✓.

## Optional — Create a Saved View

After producing the report, if any violations were found, offer to create a CAST Imaging saved view for each check. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the violating objects?
- Check 1 — Controller → Repository: N objects
- Check 2 — Entity at REST: N objects
- Check 3 — Entity with Jackson: N objects
Type yes or specify which check(s) to include.
```

For each check the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="Layering — <CheckName>"`, and `object_ids` set to the IDs of the violating objects collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
