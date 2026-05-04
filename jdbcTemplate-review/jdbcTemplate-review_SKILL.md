---
name: jdbcTemplate-review
description: Detect and report all uses of Spring JdbcTemplate and NamedParameterJdbcTemplate in an application using CAST Imaging.
---

# JdbcTemplate Usage Review

Detect and report all uses of Spring's `JdbcTemplate` and `NamedParameterJdbcTemplate` in an application using the MCP imaging tool.

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
      **Filename:** `<app>-jdbcTemplate-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "jdbcTemplate-review",
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

   This applies to all paginated MCP calls — `mcp__imaging__objects`, `mcp__imaging__object_details` (large `incoming_calls`), `mcp__imaging__source_files`, etc. If a result has `metadata.has_next: true`, fetch all pages, merge items, then write the combined result to the cache file.

3. **Search for JdbcTemplate usage** — run the following queries in parallel using `mcp__imaging__objects` (read full content, not count-only):

   - **JdbcTemplate fields**: `filters="fullname:contains:JdbcTemplate"`
   - **NamedParameterJdbcTemplate fields**: `filters="fullname:contains:NamedParameterJdbc"`
   - **Source files referencing jdbc**: `mcp__imaging__source_files` with `file_path="JdbcTemplate"`

   > **Note on `fullname:contains:JdbcTemplate`:** This will match any object whose fully-qualified name contains the substring — fields, methods, constructors, and class definitions. Filter the results to retain only objects that are application code (exclude objects whose `filePath` contains `JavaExtractedFiles` or `§<LISA>§`, which are framework-extracted stubs, not app code).

4. **Classify each result** — for each application-code object found:

   - **Class**: the class that declares the field or uses the template
   - **Field name**: the injected field name (typically `jdbcTemplate` or `namedParameterJdbcTemplate`)
   - **Type**: `JdbcTemplate` or `NamedParameterJdbcTemplate`
   - **File path**: the source file (translated per step 2)

   If no application-code objects are found, record zero and proceed to the report with a clean result.

5. **Check injection style** — for each class that has a JdbcTemplate field, call `mcp__imaging__source_file_details` with the translated file path and inspect whether injection is:
   - `@Autowired` field injection
   - Constructor injection (preferred)
   - `@Bean` / programmatic construction

6. **Produce the report** in this structure:

```
## JdbcTemplate Usage Report

### Summary
| Metric | Count |
|--------|-------|
| Classes using JdbcTemplate | N |
| Classes using NamedParameterJdbcTemplate | N |
| Total JdbcTemplate injection points | N |

### Finding
[If none found:]
No JdbcTemplate or NamedParameterJdbcTemplate usage detected in application code — ✓
The application uses JPA/Hibernate exclusively for data access.

[If found:]
| Class | Field | Type | Injection Style | File |
|-------|-------|------|----------------|------|
| ClassName | fieldName | JdbcTemplate / NamedParameterJdbcTemplate | @Autowired / constructor / @Bean | path/to/File.java |

### Assessment

**JdbcTemplate vs JPA coexistence:**
- Mixed JPA + JdbcTemplate usage requires careful transaction boundary management. Both must share the same `DataSource` and participate in the same `@Transactional` context.
- JdbcTemplate bypasses the JPA first-level cache (persistence context). A JDBC write followed by a JPA read within the same transaction may return stale cached data.
- JPA entity lifecycle events (`@PrePersist`, `@PostUpdate`, etc.) are not fired for rows written via JdbcTemplate.

**When JdbcTemplate is appropriate:**
- Bulk insert/update operations where JPA's per-entity overhead is prohibitive
- Complex native SQL that cannot be expressed as JPQL or Criteria API
- Read-only reporting queries against aggregate views

**When to prefer JPA instead:**
- Single-entity CRUD — always use repositories
- Any operation that should participate in JPA's optimistic locking or version-based concurrency

[If found, add:]
**Recommendation:** Audit each usage site above and confirm it has a documented justification for bypassing JPA. Where no justification exists, migrate to a Spring Data JPA repository or `@Query` with a native query.
```

Always report both the presence and absence of JdbcTemplate — a clean result should say so explicitly.

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | JdbcTemplate / NamedParameterJdbcTemplate usage | N class(es) | ℹ Justify / ✓ None |
| 2 | Injection style (field injection vs constructor) | N class(es) with field injection | ⚠ Improve / ✓ None |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | JPA + JdbcTemplate coexistence without justification | N class(es) | Document reasons; migrate to JPA where possible |
| 2 | Field injection (`@Autowired`) | N class(es) | Replace with constructor injection |
| 3 | `JdbcTemplate` used for simple CRUD | N class(es) | Migrate to a Spring Data JPA repository |

> Replace N values above with actual counts from the analysis. Remove rows where count is 0 and severity is ✓.

## Optional — Create a Saved View

After producing the report, if any JdbcTemplate usages were found in application code, offer to create a CAST Imaging saved view of those objects. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the JdbcTemplate usage sites?
- JdbcTemplate usages: N classes
Type yes to create the view.
```

If the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="JdbcTemplate Usages"`, and `object_ids` set to the IDs of the classes collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
