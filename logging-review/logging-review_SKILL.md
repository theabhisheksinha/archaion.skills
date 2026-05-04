---
name: logging-review
description: Detect and report all logging framework usage in an application using CAST Imaging. Covers SLF4J, Commons Logging, JUL, Log4j, Logback, MDC usage, and console/exception anti-patterns.
---

# Logging Usage Review

Detect and report all logging framework usage in an application using the MCP imaging tool. Covers logging APIs, MDC usage, and console/exception anti-patterns. For each check reports: detected (yes/no), and count of distinct application code objects referencing it.

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
      **Filename:** `<app>-logging-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "logging-review",
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

3. **Discover stub object IDs** — run the following `mcp__imaging__objects` queries in parallel to locate the framework stub objects. These return stubs in `JavaExtractedFiles` — do **not** count them as application usage. Their purpose is only to extract the object `id` for use in step 4.

   > **Why this two-step approach:** `fullname:contains:X,type:contains:field` matches the field's own fully-qualified name, not its declared type. A field named `LOG` of type `org.slf4j.Logger` has fullname `com.example.MyClass.LOG` — the filter never matches `org.slf4j`. The only reliable way to find application callers is to get the stub ID and then ask who calls it via `object_details` with `focus=inward`.

   | Target | Query |
   |--------|-------|
   | SLF4J LoggerFactory.getLogger | `filters="fullname:contains:org.slf4j.LoggerFactory"` |
   | SLF4J MDC | `filters="fullname:contains:org.slf4j.MDC"` |
   | Apache Commons Log/LogFactory | `filters="fullname:contains:org.apache.commons.logging.Log"` |
   | JUL Logger | `filters="fullname:contains:java.util.logging.Logger"` |
   | Log4j 1.x Logger | `filters="fullname:contains:org.apache.log4j.Logger"` |
   | Log4j 1.x MDC | `filters="fullname:contains:org.apache.log4j.MDC"` |
   | Log4j 2.x Logger | `filters="fullname:contains:org.apache.logging.log4j.Logger"` |
   | Log4j 2.x ThreadContext (MDC) | `filters="fullname:contains:org.apache.logging.log4j.ThreadContext"` |
   | Logback | `filters="fullname:contains:ch.qos.logback"` |
   | PrintStream.println (String) | `filters="fullname:contains:java.io.PrintStream.println"` |
   | PrintStream.println (boolean) | `filters="fullname:contains:java.io.PrintStream.println"` |
   | PrintStream.print | `filters="fullname:contains:java.io.PrintStream.print"` |
   | PrintStream.printf | `filters="fullname:contains:java.io.PrintStream.printf"` |
   | Throwable.printStackTrace | `filters="fullname:contains:printStackTrace"` |

   Additionally, run in parallel:
   - **@Slf4j annotation (Lombok)**: `filters="annotation:contains:Slf4j"` — this one does work directly; non-stub results are app classes using Lombok's `@Slf4j`.

   For each query: if no objects are returned, skip that target in step 4. If only stubs are returned, record their `id` values for step 4.

4. **Find application callers** — for each stub ID collected in step 3, call `mcp__imaging__object_details` with `focus=inward` in parallel. Collect the `incoming_calls` list — these are the actual application-code objects that reference the stub.

   - If `incoming_calls` is empty → the library is on the classpath but not used by application code.
   - If `incoming_calls` has entries → count them (paginate if `has_next: true`) and note the caller types (fields, methods, classes).
   - **Note:** `PrintStream.println` covers both `System.out.println` and `System.err.println` — the imaging tool does not distinguish between the two streams at the method level.

5. **Classify MDC usage** — if any MDC `incoming_calls` has application callers, identify the library from the stub's fullname:
   - `org.slf4j.MDC` → SLF4J MDC
   - `org.apache.log4j.MDC` → Log4j 1.x MDC
   - `org.apache.logging.log4j.ThreadContext` → Log4j 2.x MDC (functional equivalent)

6. **Produce the report** in this structure:

```
## Logging Usage Report

### SLF4J
| Check | Detected | App-code count |
|-------|----------|---------------|
| Logger field — direct API (LoggerFactory.getLogger callers) | Yes / No | N fields |
| @Slf4j annotation (Lombok) | Yes / No | N classes |

[If none detected:] Not used — ✓

[If Logger fields found, note field names observed (LOG, LOGGER, log, logger, etc.)]

### Apache Commons Logging
| Check | Detected | App-code count |
|-------|----------|---------------|
| Log / LogFactory callers | Yes / No | N |

[If none detected:] Not used — ✓

### Java Util Logging (JUL)
| Check | Detected | App-code count |
|-------|----------|---------------|
| java.util.logging.Logger callers | Yes / No | N |

[If none detected:] Not used — ✓

### Log4j 1.x
| Check | Detected | App-code count |
|-------|----------|---------------|
| org.apache.log4j.Logger callers | Yes / No | N |

[If none detected:] Not used — ✓

### Log4j 2.x
| Check | Detected | App-code count |
|-------|----------|---------------|
| org.apache.logging.log4j.Logger callers | Yes / No | N |

[If none detected:] Not used — ✓

### Logback
| Check | Detected | App-code count |
|-------|----------|---------------|
| ch.qos.logback callers | Yes / No | N |

[If none detected:] Not used — ✓

### MDC Usage
| MDC Class | Library | Detected | App-code count |
|-----------|---------|----------|---------------|
| org.slf4j.MDC | SLF4J | Yes / No | N |
| org.apache.log4j.MDC | Log4j 1.x | Yes / No | N |
| org.apache.logging.log4j.ThreadContext | Log4j 2.x | Yes / No | N |

[If none detected:] MDC not used — ✓

### Console Output (anti-pattern)
| Check | Detected | Calling methods |
|-------|----------|----------------|
| PrintStream.println (String) | Yes / No | N |
| PrintStream.println (boolean) | Yes / No | N |
| PrintStream.print | Yes / No | N |
| PrintStream.printf | Yes / No | N |

[If found, list the distinct calling method names]

[If none detected:] No console output detected — ✓

> Console output bypasses the logging framework entirely: output cannot be filtered by level,
> enriched with MDC context, routed to appenders, or suppressed in production without code changes.
> Note: PrintStream.println covers both System.out and System.err — the imaging tool does not
> distinguish between the two streams at the method level.

### printStackTrace() (anti-pattern)
| Check | Detected | Calling methods |
|-------|----------|----------------|
| Throwable.printStackTrace() | Yes / No | N |

[If found, list the distinct calling method names]

[If none detected:] No printStackTrace() detected — ✓

> `printStackTrace()` writes to `System.err` and loses all logging context (MDC, correlation IDs,
> log level). Always replace with `log.error("message", exception)` to keep the stack trace in
> the structured log stream.

### Summary
| Framework / Check | Detected | App-code count |
|-------------------|----------|---------------|
| SLF4J (direct Logger) | Yes / No | N |
| SLF4J (@Slf4j Lombok) | Yes / No | N |
| Apache Commons Logging | Yes / No | N |
| Java Util Logging | Yes / No | N |
| Log4j 1.x | Yes / No | N |
| Log4j 2.x | Yes / No | N |
| Logback | Yes / No | N |
| MDC (any lib) | Yes / No | N |
| System.out/err println (anti-pattern) | Yes / No | N |
| printStackTrace() (anti-pattern) | Yes / No | N |
```

Always report every section. A section with no findings must say "Not used — ✓" rather than be omitted.

> **Note on counts:** "App-code count" is the number of distinct code objects (fields, methods, classes) in the application source that reference the logging API — not the number of individual log statements. It is a usage breadth indicator, not a call frequency count.

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | SLF4J (recommended standard API) | N object(s) | ℹ Good practice / ✓ Not used |
| 2 | Apache Commons Logging | N object(s) | ℹ Acceptable / ✓ Not used |
| 3 | Java Util Logging (JUL) | N object(s) | ⚠ Discouraged with Spring / ✓ Not used |
| 4 | Log4j 1.x (EOL) | N object(s) | ⚠ Critical — CVE + EOL / ✓ Not used |
| 5 | Log4j 2.x | N object(s) | ℹ Acceptable / ✓ Not used |
| 6 | Logback direct (without SLF4J) | N object(s) | ⚠ Strong coupling / ✓ Not used |
| 7 | MDC (any framework) | N object(s) | ℹ Good practice if SLF4J / ✓ Not used |
| 8 | Console output — `System.out/err.println` | N method(s) | ⚠ Anti-pattern / ✓ None |
| 9 | `printStackTrace()` | N method(s) | ⚠ Anti-pattern / ✓ None |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | Log4j 1.x in production | N object(s) — EOL 2015, unpatched CVEs | Migrate to SLF4J + Logback or Log4j 2.x immediately |
| 2 | `printStackTrace()` | N method(s) — MDC context lost | Replace with `log.error("message", exception)` |
| 3 | `System.out/err.println` | N method(s) — bypasses the logging framework | Replace with appropriate SLF4J calls |
| 4 | Logback direct (without SLF4J facade) | N object(s) — coupled to implementation | Import `org.slf4j.Logger` instead |
| 5 | JUL (java.util.logging) | N object(s) — limited Spring integration | Migrate to SLF4J with the `jul-to-slf4j` bridge |
| 6 | Multiple coexisting frameworks | N frameworks — inconsistent output | Standardize on SLF4J + a single implementation |

> Replace N values above with actual counts from the analysis. Remove rows where count is 0 and severity is ✓.

## Optional — Create a Saved View

After producing the report, if any findings were detected, offer to create a CAST Imaging saved view for each finding group. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the objects involved?
- Console output (System.out/err): N calling methods
- printStackTrace(): N calling methods
Type yes or specify which group(s) to include.
```

For each group the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="Logging — <GroupName>"`, and `object_ids` set to the IDs of the calling methods collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
