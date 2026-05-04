---
name: modules-analysis
description: Analyze the architectural module structure of a Spring application using CAST Imaging. Reports module inventory, dependency graph, cycles, orphan modules, coupling metrics, layer violations, and Maven versioning.
---

# Module Structure Analysis

Analyze the architectural module structure of a Spring application using CAST Imaging. Reports the module inventory, inter-module dependency graph, and four checks: dependency cycles, orphan modules, coupling metrics, and layer violations.

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
      **Filename:** `<app>-modules-analysis-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "modules-analysis",
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

   This applies to all paginated MCP calls — `mcp__imaging__objects`, `mcp__imaging__architectural_graph`, `mcp__imaging__packages`, `mcp__imaging__package_interactions`, etc. If a result has `metadata.has_next: true`, fetch all pages, merge items, then write the combined result to the cache file.

3. **Fetch the architectural graph** — call `mcp__imaging__architectural_graph` with `application=<name>`. This returns the pre-built CAST Imaging view of the application's architectural nodes (modules/packages) and the directed edges between them.

   If the graph contains named architectural nodes (e.g., `order`, `payment`, `inventory`), use those directly as **modules**. Record each node's `id`, `name`, and object count.

   If the graph returns only raw package paths (e.g., `com.example.order.service`), group them into logical modules by truncating to the meaningful depth. Default grouping strategy:
   - Drop the common root prefix shared by all packages (e.g., `com.example`)
   - The next segment becomes the module name (e.g., `order`, `payment`, `infrastructure`)
   - Confirm the grouping with the user if the result is ambiguous (> 20 distinct top-level segments may indicate depth is too shallow)

   Build:
   - `modules`: map of `moduleName → {id or packageIds[], objectCount, subPackages[]}`
   - `edges`: list of `{source: moduleName, target: moduleName, linkCount}` — directed, source depends on target

4. **Supplement with `mcp__imaging__packages`** — call `mcp__imaging__packages` with `application=<name>` to get the full package list with object counts. Use this to fill in object counts if the architectural graph omitted them, and to verify the module grouping covers all packages.

   Flag any packages that did not map to a module (orphaned packages with no group) as **unclassified** in the report.

5. **Fetch detailed dependency data** — for each pair of modules that have an edge in step 3, call `mcp__imaging__package_interactions` with the relevant package IDs to get the detailed link breakdown (call counts by link type: CALL, USE, THROW, etc.). Aggregate by module pair.

   If `mcp__imaging__package_interactions` is not available or returns no data, fall back to `mcp__imaging__objects_relationships` with the package IDs of each module pair.

   Store the complete weighted dependency graph:
   - `adjacency[A]` = list of modules A depends on (outgoing)
   - `reverse_adjacency[B]` = list of modules that depend on B (incoming)

6. **Compute coupling metrics** — for each module, calculate:

   | Metric | Formula | Meaning |
   |--------|---------|---------|
   | **Ca** (afferent coupling) | `\|reverse_adjacency[M]\|` | How many other modules depend on M |
   | **Ce** (efferent coupling) | `\|adjacency[M]\|` | How many modules M depends on |
   | **N** (instability) | `Ce / (Ca + Ce)` | 0 = maximally stable, 1 = maximally unstable |

   > **Interpretation:**
   > - N ≈ 0 (stable): many modules depend on this one, it depends on few. Expected for `domain`, `model`, `common`, `core`.
   > - N ≈ 1 (unstable): nothing depends on this module, it depends on many. Expected for `web`, `controller`, `api`, `config`.
   > - **Violation:** a module whose name suggests it should be stable (domain/core/model) has N > 0.6, OR a module whose name suggests it should be unstable (controller/web/api) has N < 0.3 (meaning other modules depend on it — a layering problem).

7. **Check 1 — Dependency cycles** — run depth-first search on the dependency graph to detect strongly connected components (SCCs) with more than one node.

   Algorithm:
   - For each unvisited module, perform DFS. Track the current DFS path.
   - If a visited module on the current path is re-encountered → cycle detected.
   - Collect all modules in the cycle.

   For each cycle found, record: the cycle path (A → B → C → A), the number of cross-module links in the cycle, file paths of the key coupling classes (from `mcp__imaging__package_interactions` results).

8. **Check 2 — Orphan modules** — flag modules where `Ca = 0` (nothing depends on them) AND the module name does not suggest it is an application entry point.

   Entry-point heuristics (do NOT flag):
   - Name contains: `main`, `app`, `application`, `launcher`, `startup`
   - Name contains: `web`, `controller`, `api`, `rest`, `endpoint`, `resource` — these are leaf nodes by design
   - Name contains: `config`, `configuration`, `setup` — cross-cutting, expected to have Ca = 0

   Flag any module with Ca = 0 that does not match any heuristic above — it may be dead code or an unused module. Also Grep `pom.xml` / `build.gradle` under the local codebase root to verify whether the module is declared as a build artifact but has no runtime consumers.

9. **Check 3 — Coupling metrics anomalies** — flag modules whose instability N is inconsistent with their expected role:

   **Unexpectedly unstable (N > 0.6 but name suggests stability):**
   - Name contains: `domain`, `model`, `entity`, `core`, `common`, `shared`, `base`, `util`
   - These should be stable foundations — high N means too many outgoing dependencies and/or nothing depends on them, suggesting the module may be poorly scoped.

   **Unexpectedly stable (N < 0.3 but name suggests instability):**
   - Name contains: `controller`, `web`, `api`, `rest`, `endpoint`, `resource`, `ui`, `presentation`
   - These should be leaf nodes — low N means other modules depend on them, a classic upward-dependency violation.

   **Highly coupled (Ce > 5):**
   - Any module with efferent coupling > 5 is a coupling hotspot — it knows about too many other modules. Flag it regardless of N.

   For flagged modules, list their top 3 most-linked dependencies (by link count from step 5).

10. **Check 4 — Layer violations** — classify each module into an architectural layer based on its name, then detect dependencies that go in the wrong direction.

    **Layer classification (highest to lowest):**
    | Layer | Name patterns (case-insensitive) |
    |-------|----------------------------------|
    | Web | `controller`, `web`, `rest`, `api`, `endpoint`, `resource`, `graphql`, `grpc` |
    | Application | `service`, `application`, `usecase`, `facade`, `orchestrat` |
    | Domain | `domain`, `business`, `core`, `model`, `entity`, `aggregate`, `valueobject` |
    | Infrastructure | `repository`, `dao`, `persistence`, `store`, `infrastructure`, `adapter`, `gateway`, `client` |
    | Cross-cutting | `config`, `configuration`, `common`, `shared`, `util`, `helper`, `base`, `security`, `logging` |
    | Unknown | anything that doesn't match above |

    **Expected dependency directions (allowed):**
    - Web → Application → Domain ✓
    - Web → Domain ✓ (direct, skipping application — acceptable)
    - Application → Infrastructure ✓
    - Domain → Infrastructure ✗ (domain should not depend on infrastructure — dependency inversion violation)
    - Infrastructure → Domain ✓ (infrastructure implements domain interfaces)
    - Anything → Cross-cutting ✓
    - Cross-cutting → Web / Application / Domain ✗

    **Violations:**
    - Application → Web (application layer should not know about web)
    - Domain → Application (domain should not depend on services)
    - Domain → Infrastructure (domain should not depend on repositories — use dependency inversion)
    - Infrastructure → Web or Application (repositories should not depend on services or controllers)
    - Cross-cutting → Web / Application / Domain / Infrastructure

    For each violation, identify the key classes driving the cross-layer dependency using `mcp__imaging__package_interactions` details.

11. **Maven build versioning** — analyze `pom.xml` files to produce a per-module Java and Spring Boot version table. This step runs independently of the imaging analysis above and requires the local codebase root to be known (from step 2).

   **A. Discover all pom.xml files:**
   `Glob **/pom.xml` from the local codebase root. Exclude any paths under `target/`, `.git/`, or `.m2/`. Each remaining file is a Maven module.

   **B. Read and parse each pom.xml:**
   For each `pom.xml`, `Read` the file and extract:
   - **Module identity:** `<artifactId>`, `<groupId>` (may be inherited), `<packaging>`
   - **Parent reference:** `<parent><artifactId>`, `<parent><version>`, `<parent><relativePath>`
   - **Java version** — check in this priority order:
     1. `<properties><maven.compiler.release>` — preferred; single value that enforces source, target, and API
     2. `<properties><maven.compiler.source>` + `<properties><maven.compiler.target>` — note if they differ
     3. `<properties><java.version>` — Spring Boot convention used by `spring-boot-starter-parent`
     4. `<build><plugins>` → `maven-compiler-plugin` `<configuration>` → `<release>` or `<source>`/`<target>`
   - **Spring Boot version** — check in this priority order:
     1. `<parent>` with `<artifactId>spring-boot-starter-parent</artifactId>` → extract `<version>`
     2. `<dependencyManagement>` → `spring-boot-dependencies` BOM → extract `<version>`
     3. `<properties><spring-boot.version>` or `<properties><spring.boot.version>`
     4. Any direct `spring-boot-starter-*` dependency version (last resort fallback)

   **C. Resolve inheritance:**
   - Match each pom's `<parent><artifactId>` to the corresponding discovered pom (use `<relativePath>` if provided, else search by artifactId among discovered files).
   - Build a parent-child tree. For each module, walk up the tree to resolve its **effective version**:
     own value → parent value → grandparent value → … → `not specified`.
   - Label each resolved value with two attributes:
     - **Origin** = `Declared` (value is explicitly set in this module's own `pom.xml`) or `Inherited` (value comes from a parent or ancestor pom). Use `—` when not specified anywhere in the chain.
     - **Source** = the specific property name or declaration that provides the value (e.g., `<java.version>`, `<maven.compiler.release>`, `<maven.compiler.source/target>`, `<parent>spring-boot-starter-parent`, or `inherited from <parentArtifactId>` when inherited). Use `—` when not specified.

   **D. Detect version issues** (flag these in the report):
   - **Not specified** — no Java version or Spring Boot version found anywhere in the inheritance chain ⚠
   - **Cross-module Java mismatch** — two or more modules resolve to different Java versions (e.g., 17 vs 11) ⚠
   - **source ≠ target** — `maven.compiler.source` and `maven.compiler.target` are different values ⚠
   - **`source`/`target` without `release`** — informational: `release` is stricter (restricts API usage to the target JDK; prevents accidentally using newer APIs with a lower target)
   - **EOL Java version** — flag < 11 as end-of-life; flag < 17 as approaching EOL (LTS support window)
   - **Spring Boot version mismatch** — modules in the same project resolve to different Spring Boot versions ⚠
   - **Spring Boot 1.x / 2.x** — flag as EOL (Spring Boot 1.x: EOL 2019; Spring Boot 2.x: EOL November 2025)

12. **Produce the report** in this structure:

```
## Module Structure Report

### Module Inventory
| Module | Layer | Objects | Ca | Ce | N (Instability) |
|--------|-------|---------|----|----|----------------|
| order | Application | 42 | 3 | 2 | 0.40 |
| payment | Application | 28 | 2 | 3 | 0.60 |
| domain | Domain | 55 | 6 | 0 | 0.00 |
| web | Web | 18 | 0 | 4 | 1.00 |
| infrastructure | Infrastructure | 33 | 3 | 2 | 0.40 |

**Total: N modules, N objects, N inter-module dependencies**

[If unclassified packages found:]
> ⚠ The following packages did not map to any module and are not included in the analysis:
> `com.example.legacy.old`, ...

---

### Check 1 — Dependency Cycles
[If none:] No dependency cycles detected — ✓

[If found:]
| Cycle | Length | Total links in cycle |
|-------|--------|----------------------|
| order → payment → order | 2 modules | 14 links |
| service → repository → domain → service | 3 modules | 7 links |

**Total: N cycle(s) involving N module(s)**

For each cycle, the key coupling classes:
> **order → payment:** `OrderService` calls `PaymentClient` (8 calls), `OrderFacade` calls `PaymentRepository` (6 calls)
> **payment → order:** `PaymentEventHandler` calls `OrderService` (14 calls) — this is the back-edge creating the cycle.

> ⚠ **Anti-pattern:** Cyclic module dependencies prevent independent compilation, deployment,
> and testing of the involved modules. Break cycles by:
> 1. Introducing a shared abstraction module (interface) that both sides depend on.
> 2. Moving the back-edge dependency to an event/message (invert the dependency).
> 3. Merging the two modules if they are truly inseparable.

---

### Check 2 — Orphan Modules (Ca = 0)
[If none:] No orphan modules detected — ✓

[If found:]
| Module | Layer | Objects | Ce | Likely status |
|--------|-------|---------|-----|---------------|
| legacy-billing | Unknown | 12 | 0 | Dead code candidate ⚠ |
| experimental-v2 | Unknown | 5 | 3 | Dead code candidate ⚠ |

**Total: N orphan module(s)**

> ⚠ These modules have no inbound dependencies and are not entry-point modules. They may be:
> - Dead code left from a removed feature
> - A new module not yet wired up
> - A module only referenced from tests (test-only consumers don't appear in production bytecode)
>
> Verify in build files and test coverage before removing.

---

### Check 3 — Coupling Anomalies
[If none:] All modules have instability consistent with their role — ✓

[If found:]

**Unexpectedly unstable (domain/core modules with N > 0.6):**
| Module | Layer | Ca | Ce | N | Top outgoing dependencies |
|--------|-------|----|----|---|---------------------------|
| domain | Domain | 1 | 4 | 0.80 | service (12), repository (8), config (3) |

**Unexpectedly stable (web/api modules with N < 0.3):**
| Module | Layer | Ca | Ce | N | Modules depending on it |
|--------|-------|----|----|---|-------------------------|
| api | Web | 4 | 1 | 0.20 | service, payment, order |

**Highly coupled (Ce > 5):**
| Module | Ce | All dependencies |
|--------|----|-----------------|
| orchestration | 7 | order, payment, inventory, shipping, domain, config, util |

> ⚠ Domain modules with high instability typically indicate that domain logic has leaked outward
> (domain classes import from services/infrastructure) or that the module grouping is too coarse.
>
> Web/API modules that other modules depend on indicate that DTOs, constants, or interfaces defined
> in the web layer are being reused — move shared contracts to a dedicated `api-contract` or `dto` module.

---

### Check 4 — Layer Violations
[If none:] All inter-module dependencies follow the expected layer direction — ✓

[If found:]
| From module | Layer | To module | Layer | Direction | Key classes | Links |
|------------|-------|-----------|-------|-----------|-------------|-------|
| domain | Domain | service | Application | Domain → Application ⚠ | `Order` → `OrderService` | 3 |
| repository | Infrastructure | web | Web | Infra → Web ⚠ | `ProductRepository` → `ProductDTO` | 7 |

**Total: N layer violation(s) across N module pair(s)**

> ⚠ **Layer violation patterns detected:**
>
> **Domain → Application:** The domain model should not depend on services. Move the dependency-
> inverting interface into the domain layer so the service implements it (Dependency Inversion Principle).
>
> **Infrastructure → Web:** Infrastructure (repositories, adapters) should not import web-layer DTOs.
> Move shared DTOs to a dedicated `model` or `contract` module that both layers can depend on.

---

### Maven Build Versioning

[If no pom.xml found:] No pom.xml files found under the local codebase root — skip this section.

#### Java Version
| Module (artifactId) | Effective version | Origin | Source | Notes |
|--------------------|------------------|---------|--------|-------|
| my-parent | 17 | Declared | `<java.version>` | — |
| order-service | 17 | Inherited | inherited from my-parent | — |
| payment-service | 11 | Declared | `<maven.compiler.source/target>` | ⚠ Mismatch vs siblings; source/target without release |
| legacy-adapter | — | — | — | ⚠ No Java version in own pom or any parent |

[If all modules have the same version:] All N modules use Java `<version>` — ✓

[If mismatches found:]
> ⚠ **Java version inconsistency detected.** Modules compiled with different Java versions can cause
> runtime `UnsupportedClassVersionError` or silent API-compatibility issues when a lower-version
> module calls APIs available only in a higher-version module.
> Define `<java.version>` (or `<maven.compiler.release>`) once in the root/parent pom and remove
> all per-module overrides unless intentional.

[If EOL version found:]
> ⚠ Java `<version>` is end-of-life (no public security updates). Upgrade to Java 21 (LTS).

[If source/target without release found:]
> ℹ `maven.compiler.source` + `maven.compiler.target` without `maven.compiler.release`: the `release`
> flag is preferred — it ensures no internal JDK APIs are used and that the bytecode is strictly
> compatible with the target version. Replace both properties with `<maven.compiler.release>`.

#### Spring Boot Version
| Module (artifactId) | Effective version | Origin | Source | Notes |
|--------------------|--------------------|---------|--------|-------|
| my-parent | 3.2.4 | Declared | `<parent>spring-boot-starter-parent` | — |
| order-service | 3.2.4 | Inherited | inherited from my-parent | — |
| payment-service | 2.7.18 | Declared | `<parent>spring-boot-starter-parent` | ⚠ Mismatch vs siblings; Spring Boot 2.x EOL Nov 2025 |
| util-lib | — | — | — | ⚠ No Spring Boot version resolvable |

[If all modules have the same version:] All N modules use Spring Boot `<version>` — ✓

[If mismatches found:]
> ⚠ **Spring Boot version inconsistency detected.** Different Spring Boot versions in the same
> multi-module project can cause transitive dependency conflicts, mismatched auto-configuration,
> and incompatible actuator endpoints. Centralise the Spring Boot version in the root parent pom.

[If EOL Spring Boot version found:]
> ⚠ Spring Boot 1.x reached end-of-life in 2019. Spring Boot 2.x reaches end-of-life November 2025.
> Upgrade to Spring Boot 3.x (requires Java 17 minimum).

---

### Summary
| Check | Findings | Assessment |
|-------|---------|------------|
| Total modules | N | — |
| Total inter-module links | N | — |
| Dependency cycles | N | ⚠ Anti-pattern / ✓ None |
| Orphan modules | N | ⚠ Review / ✓ None |
| Coupling anomalies | N | ⚠ Review / ✓ None |
| Layer violations | N | ⚠ Anti-pattern / ✓ None |
| Java version — modules missing version | N | ⚠ / ✓ All specified |
| Java version — cross-module mismatch | Yes / No | ⚠ / ✓ Aligned |
| Java version — EOL | Yes / No | ⚠ / ✓ |
| Spring Boot version — modules missing version | N | ⚠ / ✓ All specified |
| Spring Boot version — cross-module mismatch | Yes / No | ⚠ / ✓ Aligned |
| Spring Boot version — EOL | Yes / No | ⚠ / ✓ |
```

Always report all four checks and the versioning section. A check with no findings must say "— ✓" rather than be omitted. If no pom.xml files are found, mark all versioning rows as "N/A — no pom.xml found".

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | Dependency cycles | N cycle(s) | ⚠ Critical / ✓ None |
| 2 | Orphan modules (Ca = 0) | N module(s) | ⚠ Review needed / ✓ None |
| 3 | Coupling anomalies | N module(s) | ⚠ Review needed / ✓ None |
| 4 | Layer violations | N violation(s) | ⚠ Critical / ✓ None |
| 5 | Java versions (inconsistencies / EOL) | N issue(s) | ⚠ / ✓ |
| 6 | Spring Boot versions (inconsistencies / EOL) | N issue(s) | ⚠ / ✓ |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | Dependency cycles | N cycle(s) between modules | Introduce a shared abstraction or invert the dependency |
| 2 | Layer violations | N direction violation(s) | Follow allowed directions (Web → App → Domain) |
| 3 | EOL versions (Java / Spring Boot) | End-of-life version(s) | Migrate to Java 21 LTS and/or Spring Boot 3.x |
| 4 | Orphan modules | N module(s) without a consumer | Verify if dead code to be removed |
| 5 | Coupling anomalies | N module(s) with inconsistent instability | Revisit the module decomposition |
| 6 | Version inconsistencies | Modules with different versions | Centralise the version in the parent pom |

> Replace N values above with actual counts from the preceding checks. Remove rows where count is 0 and severity is ✓.

## Optional — Create a Saved View

After producing the report, if any violations were found, offer to create a CAST Imaging saved view per check.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the violating modules?
- Check 1 — Dependency cycles: N modules in cycles
- Check 2 — Orphan modules: N modules
- Check 3 — Coupling anomalies: N modules
- Check 4 — Layer violations: N modules (source + target)
Type yes or specify which check(s) to include.
```

For each check the user confirms:
1. Collect the object IDs of the relevant packages/modules from the data gathered in steps 3–10.
2. Call `mcp__imaging__views` with `focus=create`, `name="Modules — <CheckName>"`, and `object_ids` set to the violating module IDs.
3. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
