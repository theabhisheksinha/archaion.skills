---
name: transactional-review
description: Detect and report all @Transactional annotation usage in an application using CAST Imaging. Classifies annotated classes and methods by architectural layer and reports self-invocation anti-patterns.
---

# @Transactional Usage Review

Detect and report all `@Transactional` annotation usage in an application using the MCP imaging tool. Classifies annotated classes and methods by architectural layer (API, service, repository, ambiguous) and reports names and counts per layer.

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
      **Filename:** `<app>-transactional-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "transactional-review",
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

3. **Fetch all @Transactional objects** — run the following two queries in parallel, reading **full content** (not count-only):

   - All annotated classes: `filters="annotation:contains:Transactional,type:contains:class"`
   - All annotated methods: `filters="annotation:contains:Transactional,type:contains:method"`

   > **False-positive caveat:** `annotation:contains:Transactional` is a substring match and will also hit `@EnableTransactionManagement` and `@TransactionalEventListener`. After receiving results, post-filter: only keep objects whose `annotations` array contains an entry that is exactly (or starts with) one of:
   > - `@Transactional`
   > - `@org.springframework.transaction.annotation.Transactional`
   > - `@javax.transaction.Transactional`
   > - `@jakarta.transaction.Transactional`
   >
   > Discard everything else as a false positive.

   Paginate if `metadata.has_next: true`.

4. **Classify each confirmed @Transactional object by layer** — for each class or method retained after step 3 post-filtering, determine its layer using the following rules **in priority order** (first match wins):

   ### Layer classification rules

   **API layer** — any of:
   - Class has `@RestController` in its `annotations`
   - Class has `@Controller` in its `annotations`
   - Class name contains `Controller` or `Resource` (case-insensitive)

   **Service / Business layer** — any of:
   - Class has `@Service` in its `annotations`
   - Class has `@Component` in its `annotations` AND name does NOT match Repository/DAO patterns
   - Class name contains `Service`, `Business`, `Manager`, `Facade`, `Processor`, `Handler` (case-insensitive)

   **Repository / DAO layer** — any of:
   - Class has `@Repository` in its `annotations`
   - Class name contains `Repository`, `Dao`, `Persistence`, `Store` (case-insensitive)

   **Ambiguous** — does not match any of the above rules.

   For **methods** annotated with `@Transactional`: classify the method based on its **parent class** (derive from `fullName` — strip the method name suffix to get the class name, then apply the same rules). If the parent class is not in the class results set, make a best-effort classification from the class name alone.

5. **Collect class names per layer** — for each layer, build the list of distinct class names (or `className.methodName` for method-level annotations) and a count.

6. **Check for self-invocation anti-pattern** — for each confirmed @Transactional **method**, call `mcp__imaging__object_details` with `focus=inward` (in parallel for all methods) to get `incoming_calls`. For each caller, check whether the caller belongs to the **same class** as the callee (compare the class prefix of the caller's `fullName` against the callee's class name).

   A same-class call is a **self-invocation anti-pattern**: Spring's AOP proxy is bypassed and the `@Transactional` annotation on the called method is silently ignored — it runs within whatever transaction context the caller already has, without applying the callee's own transaction settings (propagation, isolation, `noRollbackFor`, `readOnly`, etc.).

   > **How to determine caller class:** strip the method name suffix from the caller's `fullName` (last dot-separated segment) to get the fully-qualified class name, then compare with the callee's class.
   >
   > If the caller's `fullName` is not available from `incoming_calls`, use `mcp__imaging__object_details` with `focus=intra` on the caller ID to resolve its `fullName`.

   Build the list of self-invocation violations: each entry is `callerMethod → calleeMethod` within the same class, plus which @Transactional attributes on the callee are bypassed.

7. **Produce the report** in this structure:

```
## @Transactional Usage Report

### Overview
| Scope | Count |
|-------|-------|
| Classes annotated @Transactional | N |
| Methods annotated @Transactional | N |
| Total @Transactional occurrences | N |
| False positives discarded (@EnableTransactionManagement etc.) | N |

---

### API Layer (Controller / Resource)
[If none:] No @Transactional found on API layer classes or methods — ✓

[If found — this is an anti-pattern:]
| Class | Scope | Occurrences |
|-------|-------|-------------|
| SomeController | Class-level | 1 |
| OtherController.someMethod | Method-level | 1 |

**Total: N occurrences across N classes**

> ⚠ **Anti-pattern:** `@Transactional` on a controller couples the HTTP layer to the transaction
> boundary. Controllers should delegate to service methods that own the transaction. A controller-level
> transaction holds a DB connection open for the full duration of the HTTP request including
> serialization, network I/O, and any pre/post-processing — leading to connection pool exhaustion
> under load.

---

### Service / Business Layer
[If none:] No @Transactional found on service layer — ✓

[If found:]
| Class | Scope | Occurrences |
|-------|-------|-------------|
| SomeService | Class-level | 1 |
| SomeService.saveOrder | Method-level | 1 |

**Total: N occurrences across N classes**

> ✓ Correct placement. Service/business classes are the appropriate owners of transaction
> boundaries in a layered architecture.

---

### Repository / DAO Layer
[If none:] No @Transactional found on repository layer — ✓

[If found:]
| Class | Scope | Occurrences |
|-------|-------|-------------|
| SomeRepository | Class-level | 1 |

**Total: N occurrences across N classes**

> Acceptable if the repository contains custom query methods that require explicit transaction
> control (e.g., bulk deletes, native queries). However, if Spring Data JPA repositories are used,
> `@Transactional` is already applied by the framework — explicit annotation is redundant and
> potentially misleading.

---

### Ambiguous (layer not identified)
[If none:] No ambiguous occurrences — ✓

[If found:]
| Class | Annotations present | Scope | Occurrences |
|-------|---------------------|-------|-------------|
| SomeClass | @Component, @Qualifier | Class-level | 1 |

**Total: N occurrences across N classes**

> These classes could not be classified by annotation or name pattern. Review manually to
> determine whether the transaction boundary is placed correctly.

---

### Self-Invocation Anti-Pattern
[If none:] No self-invocation of @Transactional methods detected — ✓

[If found:]
| Caller | Callee | Bypassed @Transactional attributes |
|--------|--------|-----------------------------------|
| ClassName.callerMethod | ClassName.calleeMethod | noRollbackFor / readOnly / propagation / (all) |

**Total: N self-invocation(s) across N classes**

> ⚠ **Anti-pattern:** When a Spring bean calls its own method directly (not through the Spring AOP proxy),
> the `@Transactional` annotation on the called method is **silently ignored**. The call executes within
> the caller's existing transaction context, ignoring the callee's declared propagation, isolation,
> `noRollbackFor`, and `readOnly` settings — with no error or warning at runtime.
>
> **Fix:** Extract the called method into a separate Spring bean and inject it. All calls to that bean go
> through the proxy, restoring the intended transactional semantics.

---

### Summary
| Category | Count | Assessment |
|----------|-------|------------|
| API (Controller) occurrences | N | ⚠ Anti-pattern / ✓ None |
| Service / Business occurrences | N | ✓ Correct placement |
| Repository / DAO occurrences | N | ✓ Acceptable / ⚠ Review |
| Ambiguous occurrences | N | Manual review needed |
| Self-invocations (proxy bypass) | N | ⚠ Anti-pattern / ✓ None |
| **Total @Transactional occurrences** | **N** | |
```

Always report all five sections (four layers + self-invocation). A section with no occurrences must say "— ✓" rather than be omitted.

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | @Transactional on API layer (Controller) | N occurrence(s) | ⚠ Anti-pattern / ✓ None |
| 2 | @Transactional on Service layer | N occurrence(s) | ✓ Correct placement |
| 3 | @Transactional on Repository / DAO layer | N occurrence(s) | ℹ Acceptable / ✓ None |
| 4 | @Transactional — ambiguous layer | N occurrence(s) | ℹ Manual review / ✓ None |
| 5 | Self-invocation (AOP proxy bypass) | N occurrence(s) | ⚠ Anti-pattern / ✓ None |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | Auto-invocation (proxy bypass) | N method(s) — @Transactional silently ignored | Extract the called method into a separate Spring bean |
| 2 | @Transactional on Controller | N occurrence(s) — transaction held open for the full HTTP cycle | Move transactional logic into the service |
| 3 | @Transactional on Repository with Spring Data JPA | N occurrence(s) — redundant annotation | Remove redundant annotations; Spring Data handles them |
| 4 | @Transactional on ambiguous class | N occurrence(s) — class role not identifiable | Clarify the architecture and class positioning |

> Replace N values above with actual counts from the preceding checks. Remove rows where count is 0 and severity is ✓.

## Optional — Create a Saved View

After producing the report, if any violations were found, offer to create a CAST Imaging saved view for each finding group. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the violating objects?
- API layer @Transactional: N objects
- Self-invocations: N objects
Type yes or specify which group(s) to include.
```

For each group the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="Transactional — <GroupName>"`, and `object_ids` set to the IDs of the violating objects collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
