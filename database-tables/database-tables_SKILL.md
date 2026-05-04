---
name: database-tables
description: Analyze the database table structure of an application using CAST Imaging. Reports independent tables (no FK links), table clusters (connected components via FK), and classifies independent tables by library/framework origin.
---

# Database Table Structure Analysis

Analyze the database table structure of an application using the MCP imaging tool. Reports independent tables (no FK links), table clusters (connected components via FK), and classifies independent tables by library/framework origin.

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
      **Filename:** `<app>-database-tables-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "database-tables",
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

   This applies to all paginated MCP calls — `mcp__imaging__objects`, `mcp__imaging__object_details` (large `incoming_calls`), `mcp__imaging__application_database_explorer`, `mcp__imaging__objects_relationships`, etc. If a result has `metadata.has_next: true`, fetch all pages, merge items, then write the combined result to the cache file.

3. **Fetch all tables** — call `mcp__imaging__application_database_explorer` with `application=<name>` and `limit=200`. Record every table's `id` and `name`. Note the total count from `metadata.total_items`.

4. **Fetch all FK relationships** — call `mcp__imaging__objects_relationships` with all table IDs collected in step 3, using `link_mode="all"`. Read the `summary.total_edges` from the result.

   > **Truncation warning:** The edges array in the response may be capped at ~100 items even if `summary.total_edges` is higher. If `summary.total_edges` > number of edges visible in the content, note the gap. Use domain knowledge and follow-up queries (step 5) to resolve ambiguous cases.

5. **Identify candidate isolated tables** — a table is a candidate isolated table if it appears in **neither** the `source` nor the `target` of any visible edge. For each candidate:

   - Call `mcp__imaging__objects_relationships` with the candidate IDs **plus a representative sample of hub tables** (the most-connected tables visible in the edge list). This scopes the query to reveal any hidden FK from candidate to hub.
   - If still no edge appears: the table is confirmed isolated.

6. **Run Union-Find (connected components)** — treating all edges as undirected, compute connected components across all 89 (or N) tables. Each component with more than one table is a **cluster**. Each component of size 1 is an **independent table**.

7. **Classify each independent table** — for each confirmed isolated table, determine its origin using the name alone:

   ### Known library/framework table names
   Check the table name against these patterns (do not check for library presence separately — derive identification from the name itself):

   | Pattern / Exact name | Library / Framework |
   |----------------------|---------------------|
   | `databasechangelog`, `databasechangeloglock` | Liquibase |
   | `flyway_schema_history` | Flyway |
   | `qrtz_*` (any name starting with `qrtz_`) | Quartz Scheduler |
   | `shedlock` | ShedLock |
   | `batch_job_*`, `batch_step_*` | Spring Batch |
   | `spring_session`, `spring_session_attributes` | Spring Session |
   | `userconnection` | Spring Social |
   | `acl_sid`, `acl_class`, `acl_object_identity`, `acl_entry` | Spring Security ACL |
   | `persistent_logins` | Spring Security — remember-me |
   | `oauth2_*` | Spring Authorization Server |

   ### Temp / backup naming
   Flag any table whose name starts or ends with: `temp`, `tmp`, `back`, `backup`, `old`, `archive`, `bak` (case-insensitive).

   ### Application-specific patterns
   If the table shares a prefix/suffix with other application tables (e.g., `sm_` in a Shopizer codebase), note it as likely application-owned with the inferred purpose.

   ### Unknown
   If no pattern matches, mark as **Unknown**.

8. **Produce the report** in this structure:

```
## Database Table Structure

### Overview
| Metric | Count |
|--------|-------|
| Total tables | N |
| Independent tables (no FK) | N |
| Table clusters | N |

### Independent Tables

| Table | Likely identification |
|-------|----------------------|
| [unknown tables, sorted A→Z] | Unknown |
| [identified tables grouped by library/framework, consecutive rows per group] | [Library/framework — purpose] |

> Temp/backup tables: [list any found, or "None detected"]

### Table Clusters

| Cluster | Tables | Key hub tables |
|---------|--------|----------------|
| Main cluster | N | table_a, table_b, table_c, … |
| Cluster 2 (if any) | N | table_x, table_y |

[For the main cluster, group member tables by domain area in a sub-table:]

#### Main Cluster — Domain Breakdown
| Domain area | Tables | Count |
|-------------|--------|-------|
| [area] | table1, table2, … | N |
```

Always report both independent tables and clusters. If all tables form one cluster, say so explicitly. If no temp/backup tables are found, state that explicitly.

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | Independent tables (no FK) | N table(s) | ℹ To classify |
| 2 | Temp/backup tables | N table(s) | ⚠ Review needed / ✓ None |
| 3 | Tables of unknown origin | N table(s) | ℹ Investigate / ✓ None |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | Temp/backup tables | N table(s) named `tmp_*`, `old_*`, etc. | Verify if still in use; delete or archive |
| 2 | Unknown isolated tables | N table(s) with no FK and no identification | Investigate their origin and usage in application code |
| 3 | Third-party library tables | N table(s) identified (Liquibase, Quartz, etc.) | Verify the corresponding libraries are properly configured |

> Replace N values above with actual counts from the analysis. Remove rows where count is 0 and severity is ✓.

## Optional — Create a Saved View

After producing the report, offer to create a CAST Imaging saved view of the tables. Table IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the tables?
- All tables: N tables
- Independent tables only: N tables
- Main cluster only: N tables
Type yes or specify which group to include.
```

For the group the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="DB Tables — <GroupName>"`, and `object_ids` set to the IDs of the relevant tables collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
