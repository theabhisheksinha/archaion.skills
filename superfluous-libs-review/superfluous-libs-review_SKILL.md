---
name: superfluous-libs-review
description: Detect and report usage of libraries superseded by modern Java or Spring Boot defaults. Checks for Gson (superseded by Jackson) and Joda Time (superseded by java.time) usage.
---

# Superfluous Libraries Review

Detect and report usage of libraries that are superseded by modern Java or Spring Boot defaults: **Gson** (superseded by Jackson, which Spring Boot includes by default) and **Joda Time** (superseded by `java.time` / JSR-310, built into Java 8+).

For each library reports: whether it is present on the classpath, how many application objects are linked to it, and which specific types are used.

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
      **Filename:** `<app>-superfluous-libs-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "superfluous-libs-review",
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

3. **Discover stub object IDs** — run all `mcp__imaging__objects` queries below in parallel. Results will be stubs in `JavaExtractedFiles` — do not count them as application usage. Record each object's `id` for step 4.

   > **Why stubs first:** `fullname:contains:X` for field-type matching does not work — see logging skill. The correct approach is to retrieve stub IDs then query inward callers via `object_details`.

   ### Gson
   - `filters="fullname:contains:com.google.gson.Gson"`
   - `filters="fullname:contains:com.google.gson.GsonBuilder"`
   - `filters="fullname:contains:com.google.gson.JsonObject"`
   - `filters="fullname:contains:com.google.gson.JsonArray"`
   - `filters="fullname:contains:com.google.gson.JsonElement"`
   - `filters="fullname:contains:com.google.gson.JsonParser"`
   - `filters="fullname:contains:com.google.gson.annotations"` ← `@SerializedName`, `@Expose`

   ### Joda Time
   - `filters="fullname:contains:org.joda.time.DateTime"`
   - `filters="fullname:contains:org.joda.time.LocalDate"`
   - `filters="fullname:contains:org.joda.time.LocalDateTime"`
   - `filters="fullname:contains:org.joda.time.LocalTime"`
   - `filters="fullname:contains:org.joda.time.Duration"`
   - `filters="fullname:contains:org.joda.time.Period"`
   - `filters="fullname:contains:org.joda.time.Instant"`
   - `filters="fullname:contains:org.joda.time.DateTimeZone"`
   - `filters="fullname:contains:org.joda.time.format"` ← `DateTimeFormat`, `DateTimeFormatter`

   If a query returns no objects at all (not even stubs), the library is absent from the classpath — skip it in step 4 and mark as "Not present".

4. **Find application callers** — for each stub ID collected in step 3, call `mcp__imaging__object_details` with `focus=inward` in parallel. Collect `incoming_calls`.

   - If `incoming_calls` is empty → library is on classpath but not used by application code.
   - If `incoming_calls` has entries → count them (paginate if `has_next: true`) and note the caller types.
   - Aggregate counts per Gson type and per Joda Time type separately.

5. **Produce the report** in this structure:

```
## Superfluous Libraries Report

### Gson
[If no stubs found at all:] Not present on classpath — ✓

[If stubs found but zero inward callers:] Present on classpath but not used by application code.
Recommendation: remove the `com.google.gson:gson` dependency — Jackson (already included via
`spring-boot-starter-web`) covers all JSON serialization needs.

[If callers found:]
| Gson Type | App-code objects linked |
|-----------|------------------------|
| com.google.gson.Gson | N |
| com.google.gson.GsonBuilder | N |
| com.google.gson.JsonObject | N |
| com.google.gson.JsonArray | N |
| com.google.gson.JsonElement | N |
| com.google.gson.JsonParser | N |
| com.google.gson.annotations | N |
| **Total** | **N** |

**Why Gson is superfluous in a Spring Boot application:**
- Spring Boot auto-configures Jackson (`ObjectMapper`) as the default JSON serializer/deserializer
  for all REST endpoints via `spring-boot-starter-web`.
- Having both Gson and Jackson on the classpath risks inconsistency: endpoints serialize via
  Jackson while manual `Gson` calls may apply different field naming, null handling, or date
  formatting rules.
- `@SerializedName` (Gson) and `@JsonProperty` (Jackson) are not interchangeable — mixing them
  leads to silent serialization bugs.
- **Recommendation:** migrate all `Gson` usages to `ObjectMapper` or `@JsonProperty` / Jackson
  annotations, then remove the `gson` dependency.

---

### Joda Time
[If no stubs found at all:] Not present on classpath — ✓

[If stubs found but zero inward callers:] Present on classpath but not used by application code.
Recommendation: remove the `joda-time` dependency — `java.time` (JSR-310) is built into Java 8+.

[If callers found:]
| Joda Time Type | App-code objects linked |
|----------------|------------------------|
| org.joda.time.DateTime | N |
| org.joda.time.LocalDate | N |
| org.joda.time.LocalDateTime | N |
| org.joda.time.LocalTime | N |
| org.joda.time.Duration | N |
| org.joda.time.Period | N |
| org.joda.time.Instant | N |
| org.joda.time.DateTimeZone | N |
| org.joda.time.format.* | N |
| **Total** | **N** |

**Why Joda Time is superfluous in Java 8+:**
- `java.time` (JSR-310) was designed by the same author (Stephen Colebourne) as a direct
  successor to Joda Time and is part of the JDK since Java 8 — no extra dependency needed.
- Joda Time itself advises migration: its website states "Note that Joda-Time is considered to
  be a largely 'finished' project. No major enhancements are planned."
- Spring Boot's Jackson configuration (`JavaTimeModule`) natively serializes/deserializes
  `java.time` types. Joda Time types require a separate `jackson-datatype-joda` module.
- **Migration mapping:**
  | Joda Time | java.time equivalent |
  |-----------|---------------------|
  | `DateTime` | `ZonedDateTime` / `OffsetDateTime` |
  | `LocalDate` | `java.time.LocalDate` |
  | `LocalDateTime` | `java.time.LocalDateTime` |
  | `LocalTime` | `java.time.LocalTime` |
  | `Duration` | `java.time.Duration` |
  | `Period` | `java.time.Period` |
  | `Instant` | `java.time.Instant` |
  | `DateTimeZone` | `java.time.ZoneId` |
  | `DateTimeFormat` / `DateTimeFormatter` | `java.time.format.DateTimeFormatter` |

---

### Summary
| Library | On classpath | App-code objects linked |
|---------|-------------|------------------------|
| Gson | Yes / No | N |
| Joda Time | Yes / No | N |
```

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | Gson (superseded by Jackson) | N application object(s) linked | ⚠ Superfluous / ✓ Absent |
| 2 | Joda Time (superseded by `java.time`) | N application object(s) linked | ⚠ Superfluous / ✓ Absent |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | Gson coexists with Jackson (Spring Boot) | N object(s) — serialization inconsistency risk | Migrate to `ObjectMapper` / `@JsonProperty` and remove the `gson` dependency |
| 2 | Joda Time on Java 8+ | N object(s) — unnecessary external dependency | Migrate to `java.time` (JSR-310); see mapping table |

> Replace N values above with actual counts from the analysis. Remove rows where count is 0 and severity is ✓.

## Optional — Create a Saved View

After producing the report, if any application callers were found for either library, offer to create a CAST Imaging saved view per library. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the objects involved?
- Gson callers: N objects
- Joda Time callers: N objects
Type yes or specify which group(s) to include.
```

For each group the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="Superfluous Libs — <LibraryName>"`, and `object_ids` set to the IDs of the calling objects collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
