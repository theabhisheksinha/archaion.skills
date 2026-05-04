---
name: full-review
description: Run all analysis skills in sequence against a single application and produce a unified HTML report with an executive summary followed by one section per skill.
---

# Full Application Review

Run all analysis skills in sequence against a single application and produce a unified report with an executive summary followed by one section per skill.

## Arguments

Parse the arguments provided by the user at invocation time (the text after `/full-review`):

| Argument | Effect |
|----------|--------|
| `en` | Produce the entire report in **English** (default if omitted) |
| `fr` | Produce the entire report in **French** — all section titles, column headers, anti-pattern descriptions, fix guidance, and narrative text must be in French. Annotation names, class names, method names, library names, and tool names remain in their original form. |
| `edit` | Before starting the analysis, present the section list (step 2) and let the user deselect sections. |

Arguments may be combined: `/full-review fr edit` runs in French with interactive section selection.

## Steps

1. **Identify the application** — if the user hasn't specified one, call `mcp__imaging__applications` to list available applications and ask which one to review. This step is performed **once** and shared by all sub-skills below.

2. **Section selection** — the default run includes all 16 sections in this order:

   | # | Skill | Focus |
   |---|-------|-------|
   | 1 | `modules-analysis` | Architectural modules, coupling, cycles, layer violations, Maven versioning |
   | 2 | `database-tables` | DB table clusters, isolated tables, FK structure |
   | 3 | `persistence-jpa-review` | JPA/Hibernate usage, N+1, fetch strategies, entity design |
   | 4 | `persistence-layer` | Persistence layer architecture: layer boundaries, N+1, transaction analysis |
   | 5 | `jdbcTemplate-review` | JdbcTemplate usage patterns and anti-patterns |
   | 6 | `jdbc-review` | Native JDBC usage and resource management |
   | 7 | `layering` | Architectural layer violations (controller/service/repo) |
   | 8 | `transactional-review` | @Transactional usage and anti-patterns |
   | 9 | `rest-api-review` | REST API URL naming and design |
   | 10 | `http-client-review` | HTTP client usage patterns |
   | 11 | `resilience` | Resilience patterns: resilience4j, Hystrix, Spring Retry |
   | 12 | `caching-review` | @Cacheable/@CacheEvict usage, cache names, eviction gaps, anti-patterns |
   | 13 | `api-security-review` | @Validated/@Valid usage, bean validation on DTOs, decorative validation |
   | 14 | `error-handling-review` | Custom exceptions, duplicates, @ControllerAdvice, polymorphic responses |
   | 15 | `superfluous-libs-review` | Unused or redundant library dependencies |
   | 16 | `logging-review` | Logging framework usage and anti-patterns |

   **If `edit` argument was provided:** present the list above and ask the user which sections to skip (e.g., "Type the numbers to exclude, or press Enter to confirm all"). Remove any excluded sections before proceeding. Save the final selection for use in step 4.

3. **Shared setup** — perform these steps **once** before running any sub-skill. Results are reused by all sub-skills:

   **A. Ask for the local codebase root:**
   Ask the user: *"Please provide the local filesystem path to the root of the source code for **<ApplicationName>** (e.g. `C:\Users\...\shopizer-main`)."*
   Store the answer as `local_root`. Do not proceed until a valid path is provided.
   Verify it exists by running `Glob <local_root>/**/pom.xml` or `Glob <local_root>/**/*.java` — if no results, warn the user and ask again.
   Save `local_root` to memory for this application so future sessions do not need to ask again.

   **B. Resolve source file paths:** with `local_root` known, translate imaging `filePath` values using these rules:
   | `filePath` format | Action |
   |-------------------|--------|
   | `§{main_sources}§/<relative>` | Replace the `§{main_sources}§` prefix with `local_root` to get the absolute path. |
   | `§<LISA>§Scr.../<file>` | Server-only decompiled JAR — skip for `Read`/`Grep`. |
   | Absolute path | Use directly if it resolves; otherwise join with `local_root`. |
   | `null` | Skip. |

   > **Test exclusion:** Never read or analyze files under `src/test`, `**/test/**`, or any path containing `/test/`. The review covers production source code only. When using `Grep` or `Glob`, exclude test directories (e.g., `--glob '!**/src/test/**'`). When an MCP result returns a `filePath` containing `/src/test/` or `/test/`, skip it.

   **C. Locate configuration files:** run in parallel from `local_root`:
   - `Glob **/application*.yml`
   - `Glob **/application*.properties`
   Store results as `config_files`.

   **D. Locate build files:** run in parallel from `local_root`:
   - `Glob **/pom.xml` (exclude paths containing `target/`)
   - `Glob **/build.gradle`
   Store results as `build_files`.

   **E. Establish the shared cache timestamp** — generate a single timestamp string in `YYYY-MM-DD_HH-mm-ss` format (e.g. `2026-03-28_14-30-00`). Store it as `cache_ts`. This timestamp is reused by all sub-skills for their cache filenames so that all cache files from a single full-review run are grouped together.

   Each sub-skill has a **Large Result Caching** section instructing it to write MCP results with >200 items to a JSON file named `<app>-<skill>-<YYYY-MM-DD_HH-mm-ss>.json`. When running inside a full-review, substitute `<YYYY-MM-DD_HH-mm-ss>` with `cache_ts` so all cache files share the same timestamp.

   At the end of the report, list all cache files that were created during the run.

4. **Execute each selected sub-skill with incremental persistence** — each skill's results are saved to a JSON progress file immediately after completion, making the review resilient to context resets.

   **A. Initialize the progress file** — at the start of step 4, create (or load) a JSON progress file:
   - Path: `./.full-review-<app>-<timestamp>.json` (same directory as the final HTML)
   - Replace spaces in `<app>` with hyphens and lowercase it.
   - If the file **already exists** (resumed session), `Read` it and skip any section whose `status` is `"completed"`. Resume from the first incomplete section.
   - If the file does **not** exist, create it with this initial structure:

   ```json
   {
     "application": "<ApplicationName>",
     "local_root": "<local_root>",
     "language": "en|fr",
     "timestamp": "<timestamp>",
     "generated_at": "<generated_at>",
     "sections_selected": [1, 2, 3, ...],
     "sections": {}
   }
   ```

   **B. For each skill** in the confirmed selection (step 2), in order:

   a. **Check progress:** if `sections["<N>"].status == "completed"` in the JSON, skip this section entirely and inform the user (e.g., "Section 3 (JPA Persistence) — already completed, skipping.").

   b. `Read` the skill file at `.claude/skills/<skill-name>/skill.md`.

   c. Follow the domain-specific steps described in that file, **skipping any step that says "Identify the application" or "Resolve source file paths"** — those were already completed in steps 1 and 3 above. Reuse the application name, resolved paths, `config_files`, and `build_files` already in context.

   d. **Skip the per-skill "Problem Summary" section** — each sub-skill skill.md ends with a summary section for standalone use. Do **not** produce it here; a single consolidated summary will be generated at the end of the global report (step 5).

   e. Collect all findings: counts by check, violating object IDs (for optional views), and the per-skill report content. For each check within the skill, record a finding entry with: `section`, `skill`, `check`, `count`, `priority` (1–5), and `recommendation`. Priority scale: 1 = critical (data loss / security), 2 = anti-pattern (silent runtime failure), 3 = violation (architecture or best practice), 4 = warning (approaching EOL / redundancy), 5 = informational.

   f. **Render the section HTML fragment** — while all findings are still in context, convert the section's report content to an HTML fragment following the rendering rules in step 5C. This fragment is the content that goes inside the `<div class="section" id="sN">` element (excluding the outer div itself).

   g. **Save to the progress file** — `Read` the current JSON file, add/update the section entry, then `Write` the updated JSON back:

   ```json
   "sections": {
     "<N>": {
       "skill": "<skill-name>",
       "title": "<Section Title>",
       "status": "completed",
       "violations": <count>,
       "warnings": <count>,
       "findings": [
         {
           "check": "<check name>",
           "count": <N>,
           "priority": <1-5>,
           "recommendation": "<one-line fix guidance>"
         }
       ],
       "object_ids": ["<id1>", "<id2>", ...],
       "html_fragment": "<rendered HTML for this section>"
     }
   }
   ```

   h. Inform the user: "Section N (<Title>) — completed and saved. (M/Total done)"

   > **Context management:** Because each section is persisted to JSON immediately after completion, context resets are safe. On resumption in a new conversation, re-run `/full-review` with the same arguments — step 4A will detect the progress file and resume from where it left off. The user only needs to provide the application name and local_root again (or these can be recalled from memory).

5. **Produce the unified HTML report from the progress file.**

   **A. Read the progress file:**
   `Read` the JSON progress file created in step 4. Verify all selected sections have `status: "completed"`. If any are incomplete, warn the user and ask whether to proceed with a partial report or complete the remaining sections first.

   **B. Get the timestamp** from the progress file's `timestamp` and `generated_at` fields (these were set when the file was created in step 4A). If they are not present (legacy file), run `date +"%Y-%m-%d_%H-%M-%S"` and `date +"%Y-%m-%d %H:%M:%S"` via Bash.

   **C. Determine the output filename:**
   `full-review-<ApplicationName>-<timestamp>.html`
   Replace spaces in the application name with hyphens and lowercase it.
   Example: `full-review-shopizer-backend-2026-03-24_14-32-05.html`
   Write the file to the current working directory.

   **D. Build the HTML document** — assemble the final HTML by:
   1. Writing the HTML header, CSS, and nav sidebar (using section titles and violation/warning counts from the JSON).
   2. Writing the executive summary table (aggregated from each section's `violations` and `warnings` counts).
   3. Writing the top-5 findings (sorted by priority across all sections' `findings` arrays).
   4. For each section in order, wrapping the `html_fragment` from the JSON inside `<div class="section" id="sN">`.
   5. Writing the consolidated summary tables (discovery order + priority order) from all sections' `findings` arrays.
   6. Closing the HTML.

   All text content (section titles, column headers, descriptions, fix guidance, narrative) must be in the language selected at invocation (`en` default, `fr` if requested). Code names, annotation names, class names, library names, and tool names are never translated.

   > **Context efficiency:** Because each section's HTML was pre-rendered and stored in the JSON during step 4, the consolidation step does not need to re-read or re-analyze any findings. It only needs to read the JSON file and assemble the fragments — this is lightweight and can complete even with minimal remaining context.

   **HTML template structure:**

   ```html
   <!DOCTYPE html>
   <html lang="en|fr">
   <head>
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <title>Full Review — <ApplicationName> — <timestamp></title>
     <style>
       /* === Reset & base === */
       *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
       body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
              font-size: 14px; line-height: 1.6; color: #1e293b; background: #f8fafc; }
       a { color: #2563eb; text-decoration: none; }
       a:hover { text-decoration: underline; }
       code, pre { font-family: 'Cascadia Code', 'Fira Code', 'Consolas', monospace; font-size: 13px; }
       pre { background: #1e293b; color: #e2e8f0; padding: 12px 16px; border-radius: 6px;
             overflow-x: auto; margin: 8px 0; }

       /* === Layout === */
       .layout { display: flex; min-height: 100vh; }
       nav { width: 240px; min-width: 240px; background: #1e293b; color: #94a3b8;
             position: sticky; top: 0; height: 100vh; overflow-y: auto;
             padding: 20px 0; flex-shrink: 0; }
       nav .nav-title { color: #f1f5f9; font-weight: 700; font-size: 13px;
                        padding: 0 20px 16px; border-bottom: 1px solid #334155; margin-bottom: 8px; }
       nav a { display: block; padding: 6px 20px; color: #94a3b8; font-size: 13px; }
       nav a:hover, nav a.active { color: #f1f5f9; background: #334155; text-decoration: none; }
       nav .nav-badge { float: right; font-size: 11px; padding: 1px 6px; border-radius: 10px; }
       nav .badge-warn { background: #dc2626; color: white; }
       nav .badge-ok { background: #16a34a; color: white; }
       main { flex: 1; padding: 32px 40px; max-width: 1100px; }

       /* === Header === */
       .report-header { margin-bottom: 32px; }
       .report-header h1 { font-size: 24px; font-weight: 700; color: #0f172a; }
       .report-meta { margin-top: 6px; color: #64748b; font-size: 13px; }
       .report-meta span { margin-right: 20px; }

       /* === Section === */
       .section { margin-bottom: 48px; }
       .section-title { font-size: 18px; font-weight: 700; color: #0f172a; padding-bottom: 8px;
                        border-bottom: 2px solid #e2e8f0; margin-bottom: 16px; }
       .section-title .section-num { color: #64748b; font-weight: 400; margin-right: 8px; }
       h3 { font-size: 15px; font-weight: 600; color: #1e293b; margin: 20px 0 8px; }
       p, li { margin-bottom: 6px; }
       ul, ol { padding-left: 20px; margin-bottom: 8px; }

       /* === Tables === */
       table { width: 100%; border-collapse: collapse; margin: 12px 0; font-size: 13px; }
       th { background: #f1f5f9; text-align: left; padding: 8px 12px;
            border-bottom: 2px solid #cbd5e1; font-weight: 600; color: #475569; }
       td { padding: 7px 12px; border-bottom: 1px solid #e2e8f0; vertical-align: top; }
       tr:hover td { background: #f8fafc; }
       tr.row-violation td { background: #fef2f2; }
       tr.row-warning td { background: #fffbeb; }
       tr.row-ok td { background: #f0fdf4; }

       /* === Badges === */
       .badge { display: inline-block; font-size: 11px; font-weight: 600;
                padding: 2px 8px; border-radius: 12px; white-space: nowrap; }
       .badge-red   { background: #fee2e2; color: #991b1b; }
       .badge-orange{ background: #ffedd5; color: #9a3412; }
       .badge-green { background: #dcfce7; color: #166534; }
       .badge-gray  { background: #f1f5f9; color: #475569; }
       .badge-blue  { background: #dbeafe; color: #1d4ed8; }

       /* === Callouts === */
       .callout { border-left: 4px solid; padding: 10px 14px; margin: 12px 0;
                  border-radius: 0 6px 6px 0; font-size: 13px; }
       .callout-warn  { border-color: #dc2626; background: #fef2f2; }
       .callout-info  { border-color: #2563eb; background: #eff6ff; }
       .callout-ok    { border-color: #16a34a; background: #f0fdf4; }
       .callout strong { display: block; margin-bottom: 4px; }

       /* === Executive summary === */
       #exec-summary table td:nth-child(3),
       #exec-summary table td:nth-child(4) { text-align: center; font-weight: 600; }
       .total-row td { font-weight: 700; background: #f1f5f9; border-top: 2px solid #cbd5e1; }
       .top-findings { background: #fffbeb; border: 1px solid #fcd34d; border-radius: 8px;
                       padding: 16px 20px; margin-top: 20px; }
       .top-findings h3 { margin-top: 0; color: #92400e; }
     </style>
   </head>
   <body>
   <div class="layout">

     <!-- Sidebar navigation -->
     <nav>
       <div class="nav-title">Full Review<br><small style="font-weight:400;color:#64748b"><ApplicationName></small></div>
       <a href="#exec-summary">Executive Summary</a>
       <a href="#s1">1. Module Structure <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s2">2. Database Tables <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s3">3. JPA Persistence <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s4">4. Persistence Layer <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s5">5. JdbcTemplate <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s6">6. JDBC <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s7">7. Layering <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s8">8. @Transactional <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s9">9. REST API <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s10">10. HTTP Clients <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s11">11. Resilience <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s12">12. Caching <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s13">13. API Security <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s14">14. Error Handling <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s15">15. Superfluous Libs <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s16">16. Logging <span class="nav-badge badge-warn|badge-ok">N</span></a>
       <a href="#s-summary" style="margin-top:12px;border-top:1px solid #334155;padding-top:12px;color:#fbbf24;font-weight:600">★ Consolidated Summary</a>
     </nav>

     <main>
       <!-- Report header -->
       <div class="report-header">
         <h1>Full Application Review — <ApplicationName></h1>
         <div class="report-meta">
           <span>📅 Generated: <generated_at></span>
           <span>🌐 Language: EN|FR</span>
           <span>📋 Sections: N/16</span>
         </div>
       </div>

       <!-- Executive summary -->
       <div class="section" id="exec-summary">
         <div class="section-title">Executive Summary</div>
         <table>
           <thead><tr><th>#</th><th>Section</th><th>Violations</th><th>Warnings</th><th>Status</th></tr></thead>
           <tbody>
             <!-- One <tr> per section. Use class="row-violation" if violations > 0,
                  "row-warning" if only warnings, "row-ok" if all clear -->
             <tr class="row-violation|row-warning|row-ok">
               <td>1</td><td>Module Structure</td>
               <td><span class="badge badge-red|badge-green">N</span></td>
               <td><span class="badge badge-orange|badge-gray">N</span></td>
               <td><span class="badge badge-red|badge-green">⚠ N issues|✓ Clean</span></td>
             </tr>
             <!-- ... repeat for all sections ... -->
             <tr class="total-row">
               <td colspan="2">Total</td>
               <td><span class="badge badge-red">N</span></td>
               <td><span class="badge badge-orange">N</span></td>
               <td></td>
             </tr>
           </tbody>
         </table>

         <!-- Top 5 cross-section findings -->
         <div class="top-findings">
           <h3>⚠ Top findings requiring immediate attention</h3>
           <ol>
             <li><strong>[SectionName]</strong> — [one-line description of the most severe finding]</li>
             <!-- up to 5 entries, sorted by severity -->
           </ol>
         </div>
       </div>

       <!-- Section 1 -->
       <div class="section" id="s1">
         <div class="section-title"><span class="section-num">01</span>Module Structure</div>
         <!-- Full content from modules-analysis skill rendered as HTML.
              Convert all markdown tables to <table> elements.
              Convert > blockquotes to <div class="callout callout-warn|callout-info|callout-ok">.
              Convert code blocks to <pre><code>.
              Convert **bold** to <strong>, `inline code` to <code>.
              Each sub-check gets an <h3> heading. -->
       </div>

       <!-- Sections 2–16: same structure, ids s2 through s16 -->

       <!-- Consolidated summary (always last) -->
       <div class="section" id="s-summary">
         <div class="section-title">★ Consolidated Problem Summary</div>

         <h3>By discovery order (order sections ran)</h3>
         <!-- One row per finding across all sections, in the order checks ran.
              Columns: Section # | Section name | Check | Occurrences | Severity badge -->
         <table>
           <thead>
             <tr>
               <th>#</th>
               <th>Section</th>
               <th>Check</th>
               <th>Occurrences</th>
               <th>Severity</th>
             </tr>
           </thead>
           <tbody>
             <!-- Populate from findings_by_section in section order.
                  Use class="row-violation" for priority 1-2, "row-warning" for priority 3-4, "row-ok" if count=0. -->
             <tr class="row-violation|row-warning|row-ok">
               <td>S#</td>
               <td>[Section name]</td>
               <td>[Check name]</td>
               <td><span class="badge badge-red|badge-orange|badge-green">N</span></td>
               <td><span class="badge badge-red|badge-orange|badge-gray">⚠ Critical|⚠ Anti-pattern|✓ None</span></td>
             </tr>
           </tbody>
         </table>

         <h3 style="margin-top:28px">By priority (most critical first)</h3>
         <!-- Same findings sorted by priority (1 = most critical), then by count descending within same priority.
              Columns: Priority | Section | Issue | Occurrences | Short recommendation -->
         <table>
           <thead>
             <tr>
               <th>Priority</th>
               <th>Section</th>
               <th>Issue</th>
               <th>Occurrences</th>
               <th>Recommendation</th>
             </tr>
           </thead>
           <tbody>
             <!-- Populate sorted by priority asc, count desc. Skip rows with count = 0. -->
             <tr class="row-violation|row-warning">
               <td><span class="badge badge-red|badge-orange">P1|P2|P3|P4</span></td>
               <td>[Section name]</td>
               <td>[Problem description]</td>
               <td><span class="badge badge-red|badge-orange">N</span></td>
               <td>[One-line fix guidance]</td>
             </tr>
           </tbody>
         </table>
       </div>

     </main>
   </div>
   </body>
   </html>
   ```

   **Rendering rules** — these are applied in step 4F when rendering each section's HTML fragment (not during consolidation):
   - Markdown tables → `<table>` with `<thead>`/`<tbody>`; rows with findings get `class="row-violation"` or `class="row-warning"`
   - `> ⚠ ...` blockquotes → `<div class="callout callout-warn">`; `> ℹ ...` → `callout-info`; `> ✓ ...` → `callout-ok`
   - ` ```java ... ``` ` code blocks → `<pre><code>`
   - Inline `code` → `<code>`
   - `⚠` symbols in table cells → `<span class="badge badge-red">⚠</span>`
   - `✓` symbols → `<span class="badge badge-green">✓</span>`
   - Finding counts → `<span class="badge badge-red">N</span>` for violations, `<span class="badge badge-orange">N</span>` for warnings

   **E. Write the file** using the `Write` tool at path `./<filename>` (current working directory). After writing, print the absolute path so the user can open it directly.

   **F. Clean up** — optionally delete the JSON progress file, or keep it for traceability. Inform the user of both the HTML path and the JSON path.

## Optional — Create Saved Views

After producing the report, offer to create CAST Imaging saved views for the findings across all sections. Object IDs were collected during the analysis — no extra queries needed.

Present the offer grouped by section:
```
Would you like me to create saved views for any findings?
[List each section that has violating objects with a count]
Type the section numbers to include, "all" for all sections, or press Enter to skip.
```

For each confirmed section, call `mcp__imaging__views` with `focus=create`, `name="Full Review — <SectionName>"`, and `object_ids` from the collected findings. Return the direct URL for each created view.
