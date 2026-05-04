---
name: rest-api-review
description: Review the REST API URLs of an application using CAST Imaging and produce a structured quality report covering casing conventions, verbs in URLs, plural/singular usage, and REST best practices.
---

# REST API URL Review

Review the REST API URLs of an application using the MCP imaging tool and produce a structured quality report covering both good practices and violations.

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
      **Filename:** `<app>-rest-api-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "rest-api-review",
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

   This applies to all paginated MCP calls — `mcp__imaging__objects`, `mcp__imaging__api_inventory`, `mcp__imaging__object_details`, etc. If a result has `metadata.has_next: true`, fetch all pages, merge items, then write the combined result to the cache file.

3. **Fetch the full API inventory** — call `mcp__imaging__api_inventory` with `application=<name>` starting at `page=1`. Note that pages beyond 1 may return empty objects due to a known pagination limitation; work with what is returned and state the sample size clearly in the report.

4. **Collect documentation statistics and classify endpoint types** — run the following queries in parallel using `mcp__imaging__objects`, using only the `metadata.total_items` value from each result (do not read full content):

   **Documentation annotations:**
   - **Total endpoint methods**: `filters="annotation:contains:Mapping,type:contains:method"` → call this **T**
   - **Swagger 2 fully documented**: `filters="annotation:contains:ApiOperation,type:contains:method"` → call this **S2**
   - **Partially documented**: `filters="annotation:contains:ApiImplicitParams,type:contains:method"` → call this **P** (partial = P − S2)
   - **Framework detection**: call `mcp__imaging__source_files` with `file_path="swagger"` and separately `file_path="openapi"` to detect which documentation framework is configured:
     - Files containing `EnableSwagger2` / `SwaggerDefinition` → **Swagger 2 (springfox)**
     - Files containing `OpenAPI` / `springdoc` → **OpenAPI 3 (springdoc)**
     - Both present → **mixed**
     - Neither → **undocumented / no framework**

   **Endpoint type classification** — the API inventory count often exceeds the `@Mapping` method count because endpoints are registered through different mechanisms. Classify by running these queries in parallel:
   - **Spring MVC annotation-based**: `filters="annotation:contains:RequestMapping,type:contains:method"` → call this **SM**
   - **JAX-RS annotation-based**: `filters="annotation:contains:javax.ws.rs,type:contains:method"` → also check `filters="annotation:contains:jakarta.ws.rs,type:contains:method"` → call this **JR**
   - **Functional endpoints (RouterFunction)**: `filters="name:contains:RouterFunction,type:contains:method"` → also check `filters="name:contains:HandlerFunction,type:contains:method"` → call this **FE**
   - **Spring Boot Actuator**: `filters="name:contains:Endpoint,type:contains:class"` → also check `filters="annotation:contains:Endpoint,type:contains:method"` → call this **AC**
   - **Servlet-based**: `filters="name:contains:HttpServlet,type:contains:class"` or `filters="annotation:contains:WebServlet,type:contains:class"` → call this **SV**
   - **Other/unclassified**: inventory total minus all classified types above → call this **OT**

   Derive the following counts:
   - **Inventory total (N)** — from `api_inventory` metadata
   - **Annotated endpoint methods (T)** — Spring MVC `@Mapping` + JAX-RS `@GET/@POST` etc.
   - **Non-annotated endpoints (N − T)** — functional, actuator, servlet, other

   **Per-endpoint metadata**: for each endpoint type query result, collect:
   - **CAST module** — the CAST Imaging module containing the file (from `imaging_objects` or `imaging_api_inventory` response fields)
   - **Java module** — the Maven/Gradle module name (derived from the source file path, e.g. `shopizer-sm-core`, `shopizer-sm-shop`)
   - **Package** — the Java package name (derived from the source file path or `fullname` field)
   
   Use `imaging_objects` with the endpoint filters and `imaging_object_details` with `focus="intra"` to retrieve the `filePath` and `fullname` for each endpoint. Parse the Java module from the Maven structure (typically the directory after `src/main/java` or from `pom.xml` location). Parse the package from the `fullname` field (everything before the class name).

   > Note: If there are many endpoints, group by CAST module and Java module first, then list individual endpoints only for modules with issues.

   **Documentation coverage:**
   - **Fully documented (Swagger 2)** = S2
   - **Partially documented** = P − S2 (has `@ApiImplicitParams` parameter hints but no `@ApiOperation` operation description)
   - **Undocumented** = T − P (no Swagger/OpenAPI annotation at all)
   - **OpenAPI 3** = run a separate query `annotation:contains:io.swagger.v3` or check source files; if framework detection found no OpenAPI 3 files, report 0
   - **Documentation coverage rate** = S2 / T × 100

   > Note: `annotation:contains:Operation` will match both `@ApiOperation` (Swagger 2) and `@Operation` (OpenAPI 3) — always use `ApiOperation` and `io.swagger.v3` as distinct filters to avoid false positives.

   > Note: The documentation coverage denominator is **T** (annotated methods), not **N** (inventory total), because non-annotated endpoints (functional, actuator) use different documentation mechanisms or are intentionally undocumented.

5. **Analyze every URL** across four dimensions:

   ### A. Casing convention
   - **Verify programmatically** — do not rely on visual inspection. Extract all static path segments from the inventory URLs (strip `{}` path parameters), then scan each segment for:
     - uppercase letters → camelCase / PascalCase violation
     - underscores (`_`) → snake_case
     - hyphens (`-`) → kebab-case
   - Determine which convention is used: lowercase, kebab-case, snake_case, camelCase, PascalCase, or mixed.
   - Report the **dominant convention** and its consistency rate.
   - List every segment and URL that deviates from the dominant convention.
   - **Fix recommendations must align with the dominant convention.** If the dominant convention is lowercase (no separators), recommend collapsing deviating segments to lowercase — do NOT recommend kebab-case, snake_case, or any other convention that would itself deviate from the dominant one. The goal is consistency with the existing codebase, not adherence to an external standard.
   - Note: if all segments are single words, state that the separator convention has not been exercised yet (multi-word segments would reveal the true convention).

   ### A2. Underscore-prefixed segments (`_*`)
   - Identify all URLs whose **last non-parameter path segment** starts with an underscore (e.g. `_search`, `_bulk`, `_count`, `_mget`, `_scroll`).
   - These are typically Elasticsearch/OpenSearch-style action endpoints or internal/framework conventions.
   - Report:
     - Total count of such URLs and list them all
     - Whether they violate REST resource naming (action encoded in path)
     - Suggested RESTful alternatives (e.g. `POST /orders/_search` → `POST /orders/search` or use query parameters)
   - **Casing statistics split**: Provide two sets of casing convention statistics:
     1. **All segments** — the full casing analysis including `_`-prefixed segments
     2. **Excluding `_`-prefixed segments** — casing analysis after removing segments that start with `_`
     This reveals whether the application's casing convention is consistent when excluding framework/internal action endpoints.

   ### B. Verbs in URLs
   REST URLs should identify resources, not actions. HTTP methods carry the action.
   - Flag any path segment that is a verb or verb-derived action word: `login`, `logout`, `register`, `refresh`, `checkout`, `init`, `reset`, `clear`, `move`, `add`, `download`, `upload`, `send`, `update`, `delete`, `search`, `exists`, `validate`, `create`, `get`, `fetch`, `process`, `execute`, `run`, etc.
   - Also flag adjectives used as toggle actions (e.g. `visible`, `active`, `enable`).
   - Also flag check/query actions encoded as sub-paths (e.g. `unique`, `exists`) that should be query parameters or use HTTP HEAD.
   - Do NOT classify verbs yet — just collect all verbs found. Classification happens after the report using the Verb Classification Guide below.

   ### C. REST best practices
   Check for the following:
   - **Plural vs singular collections**: collection endpoints should use plural nouns (`/customers`, not `/customer`). Flag singular collections and note inconsistencies between related endpoints.
   - **Versioning**: `api/v1/` or similar versioning in the path is good practice — note if it's consistently applied.
   - **Access control / visibility encoded in URL path**: prefixes like `/auth/`, `/private/`, `/public/`, `/admin/` used to segment authorization scope in the URL are an anti-pattern. Authorization should be handled via headers/middleware. Report how many endpoints are affected.
   - **Deep nesting**: more than 3 levels of nested resources (excluding version prefix) is a smell. Flag deep paths.
   - **Sensitive data in URL**: path parameters containing passwords, tokens, or secrets should be in the request body, never the URL.
   - **Trailing slash consistency**: note whether trailing slashes are uniformly present or absent.
   - **Resource naming conflicts**: same path serving both collection and single-resource semantics with different singular/plural spellings (e.g. `POST /catalog/` vs `GET /catalogs/`).

6. **Produce the report** in this structure:

```
## API Documentation Coverage

### Framework
[Swagger 2 (springfox) | OpenAPI 3 (springdoc) | mixed | none — with evidence: source files found]

### Statistics

#### Endpoint Types
| Type | Count | % of inventory (N) | CAST Module(s) | Java Module(s) | Package(s) |
|------|-------|---------------------|----------------|----------------|------------|
| Total API endpoints (inventory) | N | 100% | — | — | — |
| Spring MVC annotated (`@RequestMapping`) | SM | SM/N% | [list] | [list] | [list] |
| JAX-RS annotated (`@GET`, `@POST`) | JR | JR/N% | [list] | [list] | [list] |
| Functional endpoints (`RouterFunction`) | FE | FE/N% | [list] | [list] | [list] |
| Spring Boot Actuator | AC | AC/N% | [list] | [list] | [list] |
| Servlet-based | SV | SV/N% | [list] | [list] | [list] |
| Other/unclassified | OT | OT/N% | [list] | [list] | [list] |

> The difference between inventory total (N) and annotated methods (SM+JR) explains why documentation coverage uses the smaller denominator.
> If an endpoint type spans multiple modules/packages, list them comma-separated. If too many, list the top 3 and note "and N more".

#### Documentation Coverage
| Metric | Count | % of annotated methods (T) |
|--------|-------|----------------------------|
| Total annotated endpoint methods | T | 100% |
| Fully documented (Swagger 2 @ApiOperation) | S2 | S2/T% |
| Fully documented (OpenAPI 3 @Operation) | O3 | O3/T% |
| Partially documented (@ApiImplicitParams only) | P−S2 | (P−S2)/T% |
| Undocumented | T−P | (T−P)/T% |
| Overall documentation coverage | S2+O3 | (S2+O3)/T% |

### Impact & Recommendation
[Based on the framework and coverage rate, state:
- If Swagger 2 only: Swagger 2 / springfox is in maintenance mode since 2020; it does not support Spring Boot 3+ or Jakarta EE. Migration to OpenAPI 3 (springdoc-openapi) is strongly recommended. Provide the maven/gradle dependency to add.
- If coverage < 100%: list the undocumented and partially documented endpoint count; explain that incomplete documentation breaks API consumers, prevents contract-first testing, and hinders auto-generated client SDKs.
- If OpenAPI 3 only and coverage = 100%: note this as a best practice.
- If no framework: explain the full risk — no discoverability, no contract enforcement, no client generation.]

## Casing Convention

### Including all segments
[dominant convention, consistency assessment, deviations if any]

### Excluding `_`-prefixed segments
[dominant convention, consistency assessment, deviations if any — compare with the "all segments" result to show whether underscore-prefixed framework endpoints skew the casing analysis]

## Underscore-Prefixed Segments (`_*`)
[table: URL | Segment | REST Violation? | Suggested Fix]
[Total count, percentage of all endpoints, assessment of whether this is an acceptable convention or a violation]

## Good Practices Observed
[bullet list of what is done well, with examples]

## Verbs in URLs

### Unnecessary verb violations
[table: URL | Verb(s) | HTTP Method | Suggested Fix]
These are actions that map directly to HTTP methods and are REST anti-patterns.

### Necessary actions (acceptable)
[table: URL | Verb(s) | Rationale]
These are inherently action-based operations that cannot be expressed as CRUD on a resource. Not violations, but noted for transparency.

## Plural/Singular Inconsistencies
[table: URL | Issue | Suggestion]

## Other REST Violations
[table: URL | Issue | Suggestion]

## Summary

### Endpoint Types
| Type | Count | CAST Module(s) | Java Module(s) | Package(s) |
|------|-------|----------------|----------------|------------|
| Total endpoints (inventory) | N | — | — | — |
| Spring MVC annotated | SM | [list] | [list] | [list] |
| JAX-RS annotated | JR | [list] | [list] | [list] |
| Functional endpoints | FE | [list] | [list] | [list] |
| Spring Boot Actuator | AC | [list] | [list] | [list] |
| Servlet-based | SV | [list] | [list] | [list] |
| Other/unclassified | OT | [list] | [list] | [list] |

### Findings
| Category | Count |
|----------|-------|
| Fully documented (Swagger 2) | S2 |
| Fully documented (OpenAPI 3) | O3 |
| Partially documented | P−S2 |
| Undocumented | T−P |
| Unnecessary verb violations | n |
| Necessary actions (acceptable) | n |
| Plural/singular inconsistencies | n |
| Access control in path (/auth/, /private/) | n |
| Underscore-prefixed segments (_*) | n |
| Other REST violations | n |

[Note sample coverage if pagination was incomplete]
```

### Verb Classification Guide

After producing the report, use this guide to classify each verb found into **unnecessary** (violation) or **necessary** (acceptable). This classification is nuanced — evaluate each verb in the context of the endpoint's purpose.

**Unnecessary verbs** — the action maps directly to an HTTP method. These are true REST anti-patterns:
- `create`, `add`, `new` → `POST /resource`
- `get`, `fetch`, `read`, `retrieve`, `find` → `GET /resource/{id}`
- `update`, `modify`, `edit`, `patch` → `PUT/PATCH /resource/{id}`
- `delete`, `remove`, `destroy`, `drop` → `DELETE /resource/{id}`
- `list`, `getAll`, `findAll` → `GET /resource`

**Necessary actions** — the operation is inherently an action that cannot be mapped to a resource noun with an HTTP method. The verb is justified when the logic of the query/operation cannot be expressed as CRUD:
- `search` — complex query with body, cannot be a `GET` with query params
- `export`, `import` — data transformation operations
- `refresh`, `reindex`, `rebuild` — infrastructure/process operations
- `checkout`, `pay`, `cancel` — workflow state transitions
- `login`, `logout`, `register`, `authenticate` — session/auth operations (though `/auth/` prefix may still be an access-control anti-pattern)
- `validate`, `verify`, `check` — when the check involves complex logic beyond a simple boolean field
- `process`, `execute`, `run`, `trigger` — when invoking a complex server-side process
- `download`, `upload`, `send`, `notify` — when the operation involves streaming or side effects beyond CRUD
- `aggregate`, `summarize`, `report` — analytics/derived data operations

**Classification rules:**
1. For each verb found, classify as **unnecessary** or **necessary** based on the lists above and the endpoint's context
2. If unnecessary, suggest the RESTful alternative using proper HTTP method + resource name
3. If necessary, note that it is acceptable but suggest dropping the `_` prefix if present (e.g. `_search` → `search`)
4. Adjectives used as toggle actions (e.g. `visible`, `active`, `enable`) are always unnecessary — they map to `PATCH` with a boolean field
5. Check/query actions encoded as sub-paths (e.g. `unique`, `exists`) are unnecessary when checking a single field — use query parameters or HTTP HEAD

Always report both good practices and issues — a "no violations found" category should say so explicitly rather than be omitted.

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | Endpoint type classification | SM + JR + FE + AC + SV + OT = N | ℹ Inventory breakdown |
| 2 | API documentation coverage (Swagger / OpenAPI) | N% coverage | ⚠ Insufficient / ✓ Complete |
| 3 | URL segment casing convention (all segments) | N violation(s) | ⚠ Inconsistent / ✓ Consistent |
| 4 | URL segment casing convention (excluding _* segments) | N violation(s) | ⚠ Inconsistent / ✓ Consistent |
| 5 | Unnecessary verb violations | N violation(s) | ⚠ REST anti-pattern / ✓ None |
| 6 | Necessary actions (acceptable) | N endpoint(s) | ℹ Acceptable design |
| 7 | Underscore-prefixed segments (_*) | N violation(s) | ⚠ Action in path / ✓ None |
| 8 | Singular resource names (instead of plural) | N violation(s) | ⚠ Inconsistent / ✓ None |
| 9 | Other REST violations (nesting, auth in URL, trailing slash…) | N violation(s) | ⚠ / ✓ None |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | Unnecessary verb violations | N URL(s) — action maps to HTTP method | Use HTTP verb + resource name (e.g., `POST /orders` instead of `/createOrder`) |
| 2 | Underscore-prefixed segments (_*) | N URL(s) — action encoded in path | Drop `_` prefix for necessary actions; use resource naming for unnecessary ones |
| 3 | Absent or partial documentation | N% coverage — API not discoverable | Migrate to OpenAPI 3 (springdoc) and annotate all endpoints |
| 4 | Singular resource names | N URL(s) — inconsistency with REST standards | Use plural for collections (`/customers` not `/customer`) |
| 5 | Framework Swagger 2 EOL | springfox — not compatible with Spring Boot 3 | Migrate to springdoc-openapi |
| 6 | Other REST violations | N miscellaneous violation(s) | See "Other violations" table for details |
| ℹ | Necessary actions (acceptable) | N URL(s) — action cannot map to resource | No action needed — these are justified by design |

> Replace N values above with actual counts from the analysis. Remove rows where count is 0 and severity is ✓.

## Optional — Create a Saved View

After producing the report, if any violations were found, offer to create a CAST Imaging saved view for each finding group. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the violating objects?
- Unnecessary verb violations: N endpoint methods
- Underscore-prefixed segments (_*): N endpoint methods
- Plural/singular inconsistencies: N endpoint methods
- Other REST violations: N endpoint methods
Type yes or specify which group(s) to include.
```

For each group the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="REST API — <GroupName>"`, and `object_ids` set to the IDs of the violating endpoint methods collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
