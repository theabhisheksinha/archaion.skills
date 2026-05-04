---
name: jdbc-review
description: Detect and report all uses of raw/native JDBC APIs (java.sql.*) in an application using CAST Imaging.
---

# Native JDBC Usage Review

Detect and report all uses of raw/native JDBC APIs (`java.sql.*`) in an application using the MCP imaging tool.

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
      **Filename:** `<app>-jdbc-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "jdbc-review",
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

3. **Search for native JDBC usage** — run the following queries in parallel using `mcp__imaging__objects` (read full content):

   - **java.sql types in app objects**: `filters="fullname:contains:java.sql"`
   - **PreparedStatement references**: `filters="fullname:contains:PreparedStatement"`
   - **Connection references**: `filters="fullname:contains:java.sql.Connection"`
   - **ResultSet references**: `filters="fullname:contains:ResultSet"`
   - **DriverManager references**: `filters="fullname:contains:DriverManager"`
   - **DataSource fields**: `filters="fullname:contains:DataSource,type:contains:field"` (to detect raw DataSource injection)

   Additionally, search source files for JDBC-related imports:
   - `mcp__imaging__source_files` with `file_path="java/sql"`
   - `mcp__imaging__source_files` with `file_path="javax.sql"`

   > **Filtering framework stubs:** The imaging tool extracts framework classes alongside application code. Exclude any result whose `filePath` contains `JavaExtractedFiles` or `§<LISA>§` — these are JDK/framework stubs, not application code. Only objects from the application's own source tree represent actual JDBC usage.

4. **Classify each application-code result** by JDBC API type:

   | JDBC API | Severity | Notes |
   |----------|----------|-------|
   | `DriverManager.getConnection()` | Critical | Direct connection acquisition; bypasses connection pooling entirely |
   | `javax.sql.DataSource` field (raw injection) | High | May bypass ORM; depends on usage |
   | `java.sql.Connection` | High | Manual connection lifecycle management |
   | `java.sql.PreparedStatement` | High | Raw SQL execution; no ORM mapping |
   | `java.sql.Statement` | High | Unparameterized SQL — SQL injection risk |
   | `java.sql.ResultSet` | Medium | Result of raw query; always accompanies Statement/PreparedStatement |
   | `java.sql.CallableStatement` | Medium | Stored procedure calls — acceptable if procedures are unavoidable |

5. **For each class found**, call `mcp__imaging__source_file_details` with the translated file path to inspect the source and confirm:
   - Whether the `java.sql` reference is in application logic vs. a configuration class (e.g., a `@Configuration` class setting up a `DataSource` bean — which is acceptable)
   - Whether `Statement` (non-prepared) is used — flag as SQL injection risk
   - Whether `DriverManager.getConnection()` is used directly — flag as connection pool bypass

6. **Produce the report** in this structure:

```
## Native JDBC Usage Report

### Summary
| JDBC API | Count | Severity |
|----------|-------|----------|
| DriverManager.getConnection() | N | Critical |
| java.sql.Connection (manual) | N | High |
| java.sql.PreparedStatement | N | High |
| java.sql.Statement (unparameterized) | N | High ⚠ SQL injection risk |
| java.sql.ResultSet | N | Medium |
| java.sql.CallableStatement | N | Medium |
| javax.sql.DataSource (raw field) | N | High |
| Total native JDBC touchpoints | N | — |

### Finding

[If none found in application code:]
No native JDBC (`java.sql.*` / `javax.sql.*`) usage detected in application code — ✓
Framework-extracted stubs (JDK classes in `JavaExtractedFiles/`) were excluded.
The application uses JPA/Hibernate (or Spring JdbcTemplate) exclusively.

[If found:]
| Class | API Used | Usage Context | Severity | File |
|-------|----------|--------------|----------|------|
| ClassName | java.sql.PreparedStatement | Business logic / Configuration | High | path/to/File.java |

### Issues Found

[For each DriverManager.getConnection() use:]
**⚠ Connection pool bypass** — `DriverManager.getConnection()` acquires a raw JDBC connection, bypassing the application's connection pool (HikariCP/Tomcat DBCP). This causes connection exhaustion under load and prevents connection reuse. Replace with a properly configured `javax.sql.DataSource` bean.

[For each Statement (non-prepared) use:]
**⚠ SQL injection risk** — `java.sql.Statement` executes SQL strings directly. If any portion of the SQL is derived from user input, this is a critical SQL injection vulnerability. Always use `PreparedStatement` with bound parameters.

[For each raw DataSource field in business logic:]
**⚠ ORM bypass** — Direct `DataSource` injection in business/service classes allows raw SQL execution that bypasses the JPA persistence context, entity lifecycle events, and optimistic locking. Migrate to Spring Data JPA repositories or `JdbcTemplate` (which at minimum provides exception translation and resource management).

### Assessment

**Why native JDBC in a JPA application is problematic:**
- **Persistence context pollution**: JPA caches entity state in the first-level cache. A JDBC write to the same table invalidates that cache silently — subsequent JPA reads within the same transaction return stale data.
- **No entity lifecycle events**: `@PrePersist`, `@PostUpdate`, audit listeners, and version-based optimistic locking are all bypassed.
- **Transaction fragmentation**: Raw JDBC must explicitly participate in the JPA transaction via `EntityManager.unwrap(Connection.class)` or Spring's `DataSourceUtils.getConnection()` — failing to do so creates split transactions.
- **No type mapping**: Developers must manually handle `java.sql.Date` ↔ `java.time` conversions, enum mapping, and custom type converters that JPA handles automatically.

**Acceptable native JDBC uses (do not flag):**
- `@Configuration` classes that construct `DataSource` beans — this is correct usage
- `CallableStatement` for stored procedures with no JPA equivalent
- Schema migration tools (`Flyway`, `Liquibase`) that run DDL via JDBC internally

**Recommendation:** Each native JDBC usage site in business/service classes above should be reviewed. Where no architectural justification exists, migrate to Spring Data JPA repositories or `JdbcTemplate` (for complex bulk SQL).
```

Always report both the presence and absence of native JDBC — a clean result should say so explicitly.

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | `DriverManager.getConnection()` | N occurrence(s) | ⚠ Critical / ✓ None |
| 2 | `java.sql.Connection` (manual management) | N occurrence(s) | High / ✓ None |
| 3 | `java.sql.PreparedStatement` | N occurrence(s) | High / ✓ None |
| 4 | `java.sql.Statement` (unparameterized) | N occurrence(s) | High ⚠ SQL injection risk / ✓ None |
| 5 | `java.sql.ResultSet` | N occurrence(s) | Medium / ✓ None |
| 6 | `javax.sql.DataSource` (raw injection) | N occurrence(s) | High / ✓ None |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | Unparameterized `Statement` | N occurrence(s) — SQL injection risk | Replace with `PreparedStatement` with bound parameters |
| 2 | `DriverManager.getConnection()` | N occurrence(s) — connection pool bypass | Use a `DataSource` bean (HikariCP) |
| 3 | Manual `Connection` / `PreparedStatement` | N occurrence(s) | Migrate to Spring Data JPA or `JdbcTemplate` |
| 4 | Raw `DataSource` in business logic | N occurrence(s) | Migrate to JPA repository or `JdbcTemplate` |
| 5 | `ResultSet` | N occurrence(s) | Always accompanies a Statement — address with item 3 |

> Replace N values above with actual counts from the analysis. Remove rows where count is 0 and severity is ✓.

## Optional — Create a Saved View

After producing the report, if any native JDBC usages were found in application code, offer to create a CAST Imaging saved view per severity group. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the native JDBC usage sites?
- Critical / High severity (DriverManager, Connection, PreparedStatement, Statement): N classes
- Medium severity (ResultSet, CallableStatement): N classes
Type yes or specify which group(s) to include.
```

For each group the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="Native JDBC — <SeverityGroup>"`, and `object_ids` set to the IDs of the relevant classes collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
