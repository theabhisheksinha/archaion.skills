---
name: persistence-layer
description: Analyze persistence layer architecture of a Spring application using CAST Imaging. Detects layer boundary violations (service-to-repository coupling, entity leaking, direct controller-to-repository calls, cross-module repository access, transaction boundary misalignment) and N+1 query patterns.
---

# Persistence Layer Architecture Review

Analyze the persistence layer architecture of a Spring application using CAST Imaging. Detects layer boundary violations and N+1 query patterns through static analysis combined with Imaging object links.

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

### Large Result Caching

   When any MCP call returns more than **200 items** (`metadata.total_items > 200`), do **not** keep the full result in the conversation context. Instead:

   1. Extract only the fields needed for analysis from each item (at minimum: `id`, `name`, `fullname`, `annotations`, `filePath`, `type`).
   2. Write the processed data to a JSON cache file in the current working directory:
      **Filename:** `<app>-persistence-layer-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "persistence-layer",
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

   This applies to all paginated MCP calls — `mcp__imaging__objects`, `mcp__imaging__object_details`, `mcp__imaging__applications_transactions`, `mcp__imaging__transaction_details`, etc. If a result has `metadata.has_next: true`, fetch all pages, merge items, then write the combined result to the cache file.

3. **Collect baseline inventories** — run all queries below in parallel using `mcp__imaging__objects`, reading **full content**:

   - **Controller classes**: `filters="annotation:contains:Controller,type:contains:class"`
     Post-filter: keep only those whose `annotations` array contains `@RestController` or `@Controller`. Discard `@ControllerAdvice`, `@RestControllerAdvice`, etc.
     Record `id`, `name`, `fullName`, `filePath`.

   - **Service classes**: `filters="annotation:contains:Service,type:contains:class"`
     Post-filter: keep only those whose `annotations` array contains exactly `@Service` or `@Service("...")`. Discard `@WebService`, `@ServiceActivator`, etc.
     Also query: `filters="name:contains:Service,type:contains:class"` and merge by `id`.
     Record `id`, `name`, `fullName`, `filePath`.

   - **Repository classes** (multiple queries, merge and deduplicate by `id`):
     - `filters="annotation:contains:Repository,type:contains:class"`
     - `filters="name:contains:Repository,type:contains:class"`
     - `filters="name:contains:Dao,type:contains:class"`
     Post-filter for annotation query: keep only `@Repository`. Discard `@EnableJpaRepositories`, etc.
     Record `id`, `name`, `fullName`, `filePath`.

   - **Entity classes**: `filters="annotation:contains:Entity,type:contains:class"`
     Post-filter: keep only those whose `annotations` array contains an entry that is exactly `@Entity` or starts with `@javax.persistence.Entity` or `@jakarta.persistence.Entity`. Discard `@EntityListeners`, `@EntityScan`, etc.
     Record `id`, `name`, `fullName`, `filePath`.

   - **@Transactional classes and methods** (two queries):
     - `filters="annotation:contains:Transactional,type:contains:class"`
     - `filters="annotation:contains:Transactional,type:contains:method"`
     Post-filter: keep only exact `@Transactional` or `@org.springframework.transaction.annotation.Transactional` or `@javax.transaction.Transactional` or `@jakarta.transaction.Transactional`. Discard `@EnableTransactionManagement`, `@TransactionalEventListener`, etc.
     Record `id`, `name`, `fullName`, `filePath`, parent class.

   > **Pagination:** If any result has `metadata.has_next: true`, fetch subsequent pages and concatenate before analysis.

4. **Classify objects by architectural layer** — for each object in the inventories, assign a layer based on annotations and naming:

   | Layer | Detection criteria |
   |-------|-------------------|
   | **Controller** | `@RestController`, `@Controller` annotation |
   | **Service** | `@Service` annotation, or class name ends with `Service`, `ServiceImpl`, `Facade` |
   | **Repository** | `@Repository` annotation, extends `JpaRepository`/`CrudRepository`/`PagingAndSortingRepository`, or class name ends with `Repository`, `Dao`, `DaoImpl` |
   | **Entity** | `@Entity` annotation |
   | **Ambiguous** | Does not clearly fit any layer — flag for manual review |

   Build lookup maps: `controller_ids`, `service_ids`, `repository_ids`, `entity_ids` for quick membership checks in later steps.

5. **Check 1: Controller → Repository direct calls** — for each controller class, call `mcp__imaging__object_details` with `focus="outward"` to get its outgoing calls.

   - For each outgoing call target, check if the target `id` is in `repository_ids`.
   - Flag every controller that calls a repository directly, bypassing the service layer.
   - **Category:** Layer Boundary Violation
   - **Why:** Controllers should delegate business logic and data access to the service layer. Direct controller-to-repository calls bypass transaction management, business validation, and create tight coupling between the API layer and the persistence layer. This makes it impossible to reuse the business logic outside of the HTTP context (e.g., in batch jobs, message handlers).

   Record: controller name, repository name, method called (if available from the call link), file paths.

6. **Check 2: Entity leaking beyond persistence layer** — for each controller class, call `mcp__imaging__object_details` with `focus="outward"` (reuse results from step 5 if already fetched).

   - Check if any outgoing call targets or return types reference objects in `entity_ids`.
   - Also read the controller source file and check for:
     - Method return types that are entity classes (e.g., `ResponseEntity<Product>`, `List<Order>`)
     - Method parameters that are entity classes (e.g., `@RequestBody Product product`)
     - Direct instantiation of entity classes in controller code
   - Additionally, check entity source files for Jackson annotations (`@JsonIgnore`, `@JsonProperty`, `@JsonManagedReference`, etc.) — this indicates the entity is being used as a DTO.
   - **Category:** Layer Boundary Violation / Design
   - **Why:** Exposing JPA entities directly in REST responses creates tight coupling between the database schema and the API contract. Any schema change (column rename, new field, relationship change) immediately affects the API. Lazy-loaded associations may trigger `LazyInitializationException` during JSON serialization. Entities with bidirectional relationships cause infinite recursion in Jackson. The fix is to use DTOs (Data Transfer Objects) or projections at the controller layer.

   Record: controller name, entity name, exposure type (return type / parameter / instantiation / Jackson annotation on entity), file paths.

7. **Check 3: Service-to-repository coupling analysis** — for each service class, call `mcp__imaging__object_details` with `focus="outward"` to get its outgoing calls.

   - Count how many distinct repositories each service calls.
   - Flag services that call more than **5 distinct repositories** as high coupling (threshold is configurable — mention it in the report).
   - Also flag services that call repositories from a **different module/package** than their own (cross-module access — see check 4).
   - **Category:** Coupling / Design
   - **Why:** A service that accesses many repositories is likely doing too much — it's a "god service" that should be split. High repository coupling makes the service hard to test (many mocks needed), hard to refactor (changes ripple across many tables), and hard to understand (too many data concerns in one place).

   Record: service name, list of repositories called, count, file path.

8. **Check 4: Cross-module repository access** — using the service-to-repository mappings from step 7:

   - Determine the module/package of each service and each repository from their `fullName` (e.g., `com.app.order.service.OrderService` → module `order`, `com.app.product.repository.ProductRepository` → module `product`).
   - Module detection heuristic: extract the 3rd or 4th segment of the package path (after `com.company`). If the project uses a flat package structure, fall back to the immediate parent package.
   - Flag services that call repositories belonging to a **different module**.
   - **Category:** Layer Boundary Violation / Modularity
   - **Why:** Cross-module repository access breaks module encapsulation. `OrderService` calling `ProductRepository` directly means the order module depends on the product module's internal persistence implementation. If the product team changes their repository, the order module breaks. The fix is to call the product module's **service** (public API), not its repository (internal implementation).

   Record: service name + module, repository name + module, file paths.

9. **Check 5: Transaction boundary misalignment** — using the `@Transactional` inventory from step 3:

   - For each `@Transactional` class or method, determine its architectural layer using the classification from step 4.
   - Flag:
     - **`@Transactional` on controller classes or methods** — transactions should not be managed at the API layer. The controller should delegate to a transactional service method.
     - **`@Transactional` on repository classes or methods** — usually redundant since Spring Data JPA repositories are already transactional by default. Explicit `@Transactional` on a repository suggests confusion about where transaction boundaries belong.
     - **Service methods calling multiple repositories without `@Transactional`** — using the outward calls from step 7, identify service methods that call 2+ repository methods but are not themselves `@Transactional`. This risks partial commits if one repository call fails.
   - **Category:** Transaction Management / Correctness
   - **Why:** Transaction boundaries should be at the **service layer** — this is the standard Spring pattern. Controllers are too high (the transaction would span HTTP serialization). Repositories are too low (each method would be its own transaction, preventing atomic multi-table operations). Service methods that orchestrate multiple repository calls without `@Transactional` risk data inconsistency: if the second repository call fails, the first one is already committed.

   Record: class/method name, layer, `@Transactional` attributes (propagation, readOnly, etc.), file path.

10. **Check 6: N+1 query pattern detection** — this check combines Imaging object links with source file analysis.

    **10a. Repository methods called inside loops:**
    - For each service method, call `mcp__imaging__object_details` with `focus="outward"` (reuse from step 7).
    - For service methods that call repository methods, read the service source file and check if the repository call is inside a `for`, `while`, `forEach`, `stream().map()`, or `stream().forEach()` loop.
    - **Why:** Calling a repository method N times in a loop generates N individual SQL queries. This is the classic N+1 pattern. The fix is to batch the IDs and use a single query with `WHERE id IN (...)` or a custom `@Query` with a collection parameter.

    **10b. Missing JOIN FETCH in custom @Query methods:**
    - For each repository class, read the source file and find methods annotated with `@Query`.
    - Check if the JPQL/HQL query loads an entity that has LAZY associations but does not use `JOIN FETCH` for those associations.
    - Cross-reference with the entity's relationship mappings: if the entity has `@OneToMany`, `@ManyToMany`, or `@ManyToOne(fetch=LAZY)` fields, and the query doesn't `JOIN FETCH` them, and the returned entity is likely to have those associations accessed (e.g., the calling service iterates over the collection), flag it.
    - **Why:** A JPQL query like `SELECT o FROM Order o` loads orders without their items. If the service then calls `order.getItems()` for each order, Hibernate fires a separate SELECT for each order's items — the classic N+1 problem. Adding `JOIN FETCH o.items` to the query loads everything in a single SQL statement.

    **10c. LAZY associations accessed in service loops:**
    - For each service method that loads entities (calls a repository `find*` method), read the service source and check if it:
      1. Loads a collection of entities (e.g., `findAll()`, `findByStatus()`)
      2. Iterates over the collection
      3. Accesses a LAZY association getter inside the loop (e.g., `order.getCustomer()`, `product.getCategories()`)
    - Cross-reference with the entity's relationship mappings to confirm the accessed field is LAZY.
    - **Why:** Each access to a LAZY association inside a loop triggers a separate SQL query. Loading 100 orders and accessing `order.getCustomer()` in a loop generates 100 additional SELECT queries (one per customer). The fix is to use `@EntityGraph` on the repository method, add `JOIN FETCH` to a custom `@Query`, or use a DTO projection that includes the needed data.

    Record: service method, repository method, pattern type (loop call / missing JOIN FETCH / lazy access in loop), entity involved, file paths.

    > **Source reading strategy:** Batch source reads by file. For each service file, extract all loop patterns and repository calls in a single pass. For each repository file, extract all `@Query` annotations in a single pass.

11. **Analyze top transactions with layering violations** — use CAST Imaging transactions to identify the most impactful data flow paths with layering issues.

    **11a. Fetch transactions:**
    - Call `mcp__imaging__applications_transactions` to list all transactions for the application.
    - For each transaction, call `mcp__imaging__transaction_details` to get the object list and call chain.

    **11b. Identify layering violations in transactions:**
    - For each transaction, walk the call chain and check for:
      - Controller → Repository direct calls (step 5 violations)
      - Entity objects appearing in controller-level nodes
      - Missing service layer in the call path (controller → repository with no service in between)
      - Cross-module repository access in the call path
    - Count the number of distinct objects in the transaction and the number of layering violations.

    **11c. Rank and select top transactions:**
    - **Top 3 by object count with violations:** Sort transactions that have at least 1 layering violation by their total object count (descending). Select the top 3. These represent the most complex data flows with architectural issues.
    - **Top 3 by cumulative violation count:** Sort transactions by total number of layering violations (descending). Select the top 3. These represent the most problematic data flows architecturally.
    - If the same transaction appears in both top-3 lists, note it but do not deduplicate — show it in both.

    Record: transaction name, type (GET/POST/etc.), object count, violation count, violations summary, URL path if available.

12. **Pre-flight checklist — complete before writing the report:**

    Before producing any output, verify that you have a real value (not "N" or "?") for each item below. If a value is missing, run the corresponding step now.

    - [ ] Controller class count — from step 3
    - [ ] Service class count — from step 3
    - [ ] Repository class count — from step 3
    - [ ] Entity class count — from step 3
    - [ ] @Transactional class/method count — from step 3
    - [ ] Controller → Repository direct calls — from step 5
    - [ ] Entity leaking violations — from step 6
    - [ ] Service-to-repository coupling analysis — from step 7
    - [ ] Cross-module repository access violations — from step 8
    - [ ] Transaction boundary misalignment violations — from step 9
    - [ ] N+1 query patterns detected — from step 10
    - [ ] Top transactions with violations — from step 11

    Every section in the report template below **must appear in the output**, even when the count is zero. Do not omit a section because it has no violations — write "No violations found — ✓" instead.

13. **Produce the report** in this structure:

```
## Persistence Layer Overview

| Metric | Count |
|--------|-------|
| Controller classes | N |
| Service classes | N |
| Repository classes | N |
| Entity classes | N |
| @Transactional annotations (classes + methods) | N |

## Layer Boundary Violations

### Check 1: Controller → Repository Direct Calls

[If none found:]
No controller-to-repository direct calls detected — ✓

[If found:]
| Controller | Repository | Method Called | Controller File | Repository File |
|-----------|-----------|-------------|----------------|----------------|
| ControllerName | RepositoryName | methodName() | path/to/Controller.java | path/to/Repository.java |

> **Why this matters:** Controllers should delegate to the service layer for data access. Direct repository
> calls bypass transaction management, business validation, and create tight coupling between the API and
> persistence layers. This prevents reuse of business logic outside the HTTP context.

### Check 2: Entity Leaking Beyond Persistence Layer

[If none found:]
No entity leaking detected — ✓

[If found:]

#### 2a. Entities in controller return types / parameters
| Controller | Method | Entity | Exposure Type | File |
|-----------|--------|--------|--------------|------|
| ControllerName | methodName() | EntityName | Return type / @RequestBody parameter | path/to/File.java |

#### 2b. Entities with Jackson annotations
| Entity Class | Jackson Annotations | File |
|-------------|-------------------|------|
| EntityName | @JsonIgnore, @JsonProperty, ... | path/to/File.java |

> **Why this matters:** Exposing JPA entities directly in REST responses couples the database schema to the
> API contract. Schema changes immediately break the API. Lazy associations cause
> `LazyInitializationException` during serialization. Bidirectional relationships cause infinite recursion.
> **Recommendation:** Use DTOs or projections at the controller layer.

### Check 3: Service-to-Repository Coupling

| Service | Repositories Called | Count | Assessment | File |
|---------|-------------------|-------|-----------|------|
| ServiceName | Repo1, Repo2, Repo3, ... | N | ✓ Normal / ⚠ High coupling (>5) | path/to/File.java |

[For services with count > 5, provide detail:]
> **Why this matters:** A service accessing many repositories is likely a "god service" doing too much.
> High coupling makes the service hard to test, hard to refactor, and hard to understand.
> **Recommendation:** Split into focused services, each responsible for a bounded context.

### Check 4: Cross-Module Repository Access

[If none found:]
No cross-module repository access detected — ✓

[If found:]
| Service | Service Module | Repository | Repository Module | File |
|---------|---------------|-----------|-------------------|------|
| ServiceName | order | RepositoryName | product | path/to/File.java |

> **Why this matters:** Cross-module repository access breaks module encapsulation. The calling module
> depends on another module's internal persistence implementation. If the target module changes its
> repository, the calling module breaks. **Recommendation:** Call the target module's **service** (public API)
> instead of its repository (internal implementation).

### Check 5: Transaction Boundary Misalignment

[If no misalignments found:]
Transaction boundaries correctly placed at service layer — ✓

[If found:]

#### 5a. @Transactional on controllers
| Controller | Class/Method | @Transactional Attributes | File |
|-----------|-------------|--------------------------|------|
| ControllerName | methodName() | propagation=REQUIRED, readOnly=false | path/to/File.java |

> **Why this matters:** Transactions at the controller layer span HTTP serialization, holding database
> connections longer than necessary. Move `@Transactional` to the service method.

#### 5b. @Transactional on repositories (redundant)
| Repository | Class/Method | @Transactional Attributes | File |
|-----------|-------------|--------------------------|------|
| RepositoryName | methodName() | readOnly=true | path/to/File.java |

> **Note:** Spring Data JPA repositories are transactional by default. Explicit `@Transactional` is
> redundant unless overriding specific attributes (e.g., `readOnly`, `timeout`).

#### 5c. Service methods calling multiple repositories without @Transactional
| Service | Method | Repositories Called | Has @Transactional? | File |
|---------|--------|-------------------|--------------------|----- |
| ServiceName | methodName() | Repo1, Repo2 | No ⚠ | path/to/File.java |

> **Why this matters:** Without `@Transactional`, each repository call runs in its own transaction. If the
> second call fails, the first is already committed — causing data inconsistency. Wrap multi-repository
> service methods in `@Transactional`.

## N+1 Query Pattern Detection

### Check 6a: Repository Methods Called Inside Loops

[If none found:]
No repository calls inside loops detected — ✓

[If found:]
| Service | Method | Loop Type | Repository Call | Entity | File |
|---------|--------|-----------|----------------|--------|------|
| ServiceName | methodName() | for / forEach / stream | repo.findById() | EntityName | path/to/File.java |

> **Why this matters:** Calling a repository N times in a loop generates N individual SQL queries. For 100
> iterations, this means 100 round-trips to the database instead of 1.
> **Recommendation:** Collect IDs and use a single `findAllById()` or `@Query("... WHERE id IN :ids")`.

### Check 6b: Missing JOIN FETCH in Custom @Query Methods

[If none found:]
No missing JOIN FETCH patterns detected — ✓

[If found:]
| Repository | Method | Query | Loaded Entity | Missing Fetch for | File |
|-----------|--------|-------|--------------|------------------|------|
| RepoName | findWithDetails() | SELECT o FROM Order o | Order | items (LAZY @OneToMany) | path/to/File.java |

> **Why this matters:** Loading entities without `JOIN FETCH` on LAZY associations means each association
> access triggers a separate query. Add `JOIN FETCH o.items` to load everything in a single SQL.

### Check 6c: LAZY Associations Accessed in Service Loops

[If none found:]
No LAZY association access in loops detected — ✓

[If found:]
| Service | Method | Entity | LAZY Field Accessed | Mapping Type | File |
|---------|--------|--------|-------------------|-------------|------|
| ServiceName | processOrders() | Order | getCustomer() | @ManyToOne (default EAGER but overridden to LAZY) | path/to/File.java |

> **Why this matters:** Each LAZY getter call inside a loop triggers a separate SELECT. Loading 100 orders
> and calling `getCustomer()` generates 100 additional queries.
> **Recommendation:** Use `@EntityGraph` on the repository method or `JOIN FETCH` in a custom query.

## Top Transactions with Layering Violations

### Top 3 by Object Count (most complex flows with violations)

[If no transactions with violations found:]
No transactions with layering violations detected — ✓

[If found:]
| # | Transaction | Type | URL Path | Object Count | Violation Count | Violations |
|---|------------|------|----------|-------------|----------------|------------|
| 1 | TransactionName | GET | /api/... | N | N | Controller→Repo, Entity leak, ... |
| 2 | TransactionName | POST | /api/... | N | N | ... |
| 3 | TransactionName | GET | /api/... | N | N | ... |

### Top 3 by Cumulative Violation Count (most architecturally problematic flows)

| # | Transaction | Type | URL Path | Object Count | Violation Count | Violations |
|---|------------|------|----------|-------------|----------------|------------|
| 1 | TransactionName | POST | /api/... | N | N | ... |
| 2 | TransactionName | GET | /api/... | N | N | ... |
| 3 | TransactionName | PUT | /api/... | N | N | ... |

> **Why these transactions matter:** These are the data flows with the most architectural issues. Fixing
> layering violations in these transactions will have the highest impact on code quality and maintainability.
> Each transaction represents a real user-facing operation — the violations compound along the call chain.

## Good Practices Observed
[Bullet list of what is done well — e.g., all repository access goes through service layer, DTOs used
consistently, @Transactional correctly placed at service layer, no cross-module repository access, etc.
If a category has no violations, state "No violations found" rather than omitting it.]

## Summary
| Category | Count | Status |
|----------|-------|--------|
| Controller → Repository direct calls | N | ⚠ / ✓ |
| Entity leaking (return types / parameters) | N | ⚠ / ✓ |
| Entity leaking (Jackson annotations on entities) | N | ⚠ / ✓ |
| Services with high repository coupling (>5) | N | ⚠ / ✓ |
| Cross-module repository access | N | ⚠ / ✓ |
| @Transactional on controllers | N | ⚠ / ✓ |
| @Transactional on repositories (redundant) | N | ℹ / ✓ |
| Service methods without @Transactional (multi-repo) | N | ⚠ / ✓ |
| N+1: Repository calls inside loops | N | ⚠ / ✓ |
| N+1: Missing JOIN FETCH | N | ⚠ / ✓ |
| N+1: LAZY access in loops | N | ⚠ / ✓ |
| Transactions with layering violations | N | ⚠ / ✓ |
```

Every section of the report must appear, even when the finding count is zero — write "No violations found — ✓" rather than omitting the section.

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | Controller → Repository direct calls | N violation(s) | ⚠ Layer violation / ✓ None |
| 2 | Entity leaking beyond persistence layer | N violation(s) | ⚠ Design / ✓ None |
| 3 | Service-to-repository high coupling (>5 repos) | N service(s) | ⚠ Coupling / ✓ None |
| 4 | Cross-module repository access | N violation(s) | ⚠ Modularity / ✓ None |
| 5 | Transaction boundary misalignment | N violation(s) | ⚠ Correctness / ✓ None |
| 6 | N+1 query patterns | N pattern(s) | ⚠ Performance / ✓ None |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | Service methods without @Transactional (multi-repo) | N method(s) — risk of partial commits | Add @Transactional to service methods calling multiple repositories |
| 2 | Controller → Repository direct calls | N violation(s) — bypasses service layer | Route through service layer |
| 3 | N+1: Repository calls inside loops | N pattern(s) — N individual queries | Batch with findAllById() or @Query with IN clause |
| 4 | N+1: LAZY access in service loops | N pattern(s) — N additional SELECTs | Use @EntityGraph or JOIN FETCH |
| 5 | Entity leaking to controllers | N violation(s) — API/schema coupling | Introduce DTOs or projections |
| 6 | Cross-module repository access | N violation(s) — broken encapsulation | Call target module's service instead |
| 7 | @Transactional on controllers | N violation(s) — transaction scope too wide | Move @Transactional to service method |
| 8 | Service high repository coupling | N service(s) — god service pattern | Split into focused services per bounded context |
| 9 | N+1: Missing JOIN FETCH | N query/queries — silent N+1 | Add JOIN FETCH to custom @Query |
| 10 | @Transactional on repositories | N instance(s) — redundant annotation | Remove unless overriding specific attributes |

> Replace N values above with actual counts from the analysis. Remove rows where count is 0 and severity is ✓.

---

## Appendix: MCP Result File Parsing

When an `mcp__imaging__objects` call returns a result too large to read inline, the tool saves it to a `.txt` file and returns the path. These files have a specific, non-standard serialization that requires two-pass parsing.

### Why standard `json.loads` fails

The outer file is valid JSON with three keys: `content` (string), `metadata` (object), `description` (string).

The `content` string is itself intended to be a JSON array, but it is **not cleanly serialized**:
- Between top-level tokens it uses actual newline characters (`\n`, 0x0A) — valid in JSON whitespace.
- Within annotation strings and some field values it uses **literal backslash + n** (`\` 0x5C + `n` 0x6E) — invalid as JSON whitespace, causing `JSONDecodeError: Expecting value`.
- Some annotation strings (e.g. file paths, column name strings) contain other non-standard backslash sequences such as `\e` or `\s` that are invalid JSON escape characters, causing `JSONDecodeError: Invalid \escape`.

### Parsing script

Use this Python snippet whenever you need to parse the full content of a saved MCP result file:

```python
import json

BACKSLASH = chr(92)          # 0x5C — avoids raw backslash in string literals
VALID_JSON_ESCAPES = set('"' + BACKSLASH + '/' + 'bfnrtu')

def parse_mcp_result_file(path):
    """Parse a saved mcp__imaging__objects result file.

    Returns (items: list[dict], metadata: dict).
    """
    with open(path, encoding='utf-8') as f:
        raw = f.read()

    outer = json.loads(raw)           # outer envelope is valid JSON
    content_str = outer['content']    # this string needs fixing before parsing

    # Pass 1: replace literal backslash+n with actual newlines so they act
    # as valid JSON whitespace between tokens.
    fixed = content_str.replace(BACKSLASH + 'n', '\n')

    # Pass 2: escape any remaining invalid backslash sequences.
    # Valid JSON escape chars: " \ / b f n r t u
    result = []
    i = 0
    while i < len(fixed):
        if fixed[i] == BACKSLASH and i + 1 < len(fixed):
            next_char = fixed[i + 1]
            if next_char not in VALID_JSON_ESCAPES:
                result.append(BACKSLASH + BACKSLASH)   # escape the stray backslash
            else:
                result.append(BACKSLASH)
            i += 1
        else:
            result.append(fixed[i])
            i += 1

    items = json.loads(''.join(result))
    return items, outer['metadata']
```

## Optional — Create a Saved View

After producing the report, if any violations were found, offer to create a CAST Imaging saved view per finding group. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the violating objects?
- Controllers calling repositories directly: N controllers + N repositories
- Entities leaking to controllers: N entities + N controllers
- Services with high coupling: N services
- Cross-module repository access: N services + N repositories
- Transaction boundary violations: N classes/methods
- N+1 query patterns: N services + N repositories
Type yes or specify which group(s) to include.
```

For each group the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="Persistence Layer — <GroupName>"`, and `object_ids` set to the IDs of the relevant objects collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
