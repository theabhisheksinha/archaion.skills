---
name: persistence-jpa-review
description: Review the JPA entity mappings of an application using CAST Imaging. Produces a structured quality report covering entity ID types, fetch strategies, relationship anti-patterns for all mapping types (@OneToMany, @ManyToOne, @OneToOne, @ManyToMany), @ElementCollection pitfalls, @MapsId misconfigurations, and Lombok anti-patterns.
---

# JPA Persistence Review

Review the JPA entity mappings of an application using the MCP imaging tool and produce a structured quality report covering entity ID types, fetch strategies, relationship mappings, @ManyToMany anti-patterns, @ElementCollection pitfalls, @MapsId misconfigurations, and Lombok anti-patterns.

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
      **Filename:** `<app>-persistence-jpa-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "persistence-jpa-review",
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

   This applies to all paginated MCP calls — `mcp__imaging__objects`, `mcp__imaging__object_details`, `mcp__imaging__application_database_explorer`, `mcp__imaging__quality_insights`, etc. If a result has `metadata.has_next: true`, fetch all pages, merge items, then write the combined result to the cache file.

3. **Collect overview statistics** — run the following queries in parallel using `mcp__imaging__objects`, reading only `metadata.total_items` from each result (do not read full content):

   - **Total entities**: `filters="annotation:contains:Entity,type:contains:class"`
   - **@OneToMany**: `filters="annotation:contains:OneToMany,type:contains:field"`
   - **@ManyToOne**: `filters="annotation:contains:ManyToOne,type:contains:field"`
   - **@OneToOne**: `filters="annotation:contains:OneToOne,type:contains:field"`
   - **@ManyToMany**: `filters="annotation:contains:ManyToMany,type:contains:field"`
   - **@ElementCollection**: `filters="annotation:contains:ElementCollection,type:contains:field"`
   - **Explicit EAGER**: `filters="annotation:contains:EAGER"`
   - **Explicit LAZY**: `filters="annotation:contains:LAZY"`
   - **@Fetch (Hibernate)**: `filters="annotation:contains:FetchMode"`
   - **@EntityGraph**: `filters="annotation:contains:EntityGraph"`

   > **Filter caveats — read before trusting any count:**
   > - All filters use **case-insensitive substring matching**. This inflates several counts:
   >   - `annotation:contains:Entity` also matches `@EntityScan`, `@ComponentScan(basePackages="...entity...")` on Spring config classes.
   >   - `annotation:contains:EAGER` / `annotation:contains:LAZY` are reliable since those substrings are unique to `FetchType`.
   > - **Do not query `@Id fields` or `@Data on entities` here** — both filters produce heavily inflated counts due to substring collisions (`@Valid` contains "id", `@Service("initializationDatabase")` contains "Data"). Those are analyzed with Python post-filtering in steps 4 and 7.
   > - If any result has `metadata.has_next: true`, fetch subsequent pages and concatenate before computing counts (see pagination note in step 5).

4. **Analyze ID types** — the `mangling` field and `signature` are **always empty for Java fields** in Imaging; do not use them to determine the Java type. Use a three-source approach: MCP for identification, then MCP + local source for type resolution.

   a. **Identify @Id fields via MCP:** Fetch `filters="annotation:contains:Id,type:contains:field"` (full content, not count-only). Parse the result file using the **MCP file parsing script** (see appendix) and post-filter in Python for items where the `annotations` array contains exactly `"@Id"` or `"@Id("` (string equality, not substring match). If the result spans multiple pages (`metadata.has_next: true`), fetch subsequent pages and concatenate before filtering.

   b. **Determine Java types — primary method (MCP `source_file_details`):** For each entity file that declares an `@Id` field, call `mcp__imaging__source_file_details` with `nature="inventory"`. The result contains two type-revealing elements:
      - **`getId()` method signature** — under `"type": "Java Method"`, look for the element named `getId`. Its `signature` field contains the return type, e.g., `"getId() return java.lang.Integer"` or `"getId() return java.lang.Long"`. This is the most reliable MCP-based source for the `@Id` Java type.
      - **`Java Instantiated Class`** — if the entity extends a generic base class (e.g., `SalesManagerEntity<K, E>`), the inventory lists a `"type": "Java Instantiated Class"` element like `"SalesManagerEntity<Integer,MerchantStore>"`. Extract the first generic parameter as a cross-check.

      > **Efficiency:** You do not need to call `source_file_details` for every entity. Group entities by their expected type (e.g., all `SalesManagerEntity<Long, ...>` entities likely share the same `Long` ID). Sample 3–5 representative files. If consistent, apply to all. If mixed, expand the sample until all distinct types are accounted for.

      > **Lombok edge case:** If the entity uses `@Getter`/`@Setter` or `@Data` at field or class level, the `getId()` method is generated at compile time and will **not** appear in the `source_file_details` inventory (Imaging indexes source-level methods only). In that case, `getId()` will be absent from the method list. Use the **`Java Instantiated Class`** generic parameter as the primary MCP source (e.g., `SalesManagerEntity<Long, Product>` → `Long`), and fall back to local source `Grep` (step 4c) to read the field declaration directly. The field declaration (`private Long id;`) is always present in the source regardless of Lombok.

   c. **Determine Java types — fallback (local source `Grep`):** If `source_file_details` is unavailable (e.g., decompiled JARs, `§<LISA>§` paths), the `getId()` method is missing (Lombok-generated or inherited), or the `Java Instantiated Class` entry is absent, fall back to reading the local source file directly. Use `Grep` with a **context window of at least 5 lines after `@Id`** (`-A 5`), because the `@Id` annotation is often followed by multiple other annotations before the field declaration line:
      ```java
      @Id                                          // line N
      @Column(name = "MERCHANT_ID", ...)           // line N+1
      @TableGenerator(name = "TABLE_GEN", ...)     // line N+2  (may span 2 lines)
      @GeneratedValue(strategy = ...)              // line N+3
      private Integer id;                          // line N+4  ← actual type is here
      ```
      The field type is on the line matching `private|protected|public?\s+(Long|Integer|int|long|String|UUID)\s+\w+`. If `@Id` is on a single-field `@Embeddable` (composite key), note the embeddable class name and inspect it separately.

      > **Why 5 lines of context:** In many codebases, `@Id` is stacked with `@Column`, `@TableGenerator`, `@GeneratedValue`, and `@SequenceGenerator` — the field declaration can be 3–5 lines below `@Id`. A 2-line context window will miss the type declaration in these cases.

   d. **Handle @MappedSuperclass inheritance:** If a `@MappedSuperclass` declares the `@Id` field (e.g., `Description`), all its `@Entity` subclasses inherit that type. Count the subclasses and attribute the inherited type to each. Do not double-count — the `@Id` is declared once in the superclass. The `source_file_details` for subclass files will NOT show a `getId()` method if it is inherited — you must check the superclass file instead.

   e. **Handle `SalesManagerEntity<K>` or similar generic base classes:** If entities extend a generic abstract class parameterized by the ID type (e.g., `SalesManagerEntity<Long, Product>`), extract the first generic parameter from the `extends` clause or from the `Java Instantiated Class` entry in `source_file_details`. Cross-check with the actual `@Id` field declaration or `getId()` return type.

   f. **Classify each @Id field** into one of: `Long` (wrapper), `Integer` (wrapper), `long` (primitive), `int` (primitive), `String`, `UUID`, composite key (`@EmbeddedId` / `@IdClass` / `@Embeddable`), or "unresolved". Report any field whose type cannot be confirmed as "unresolved".

5. **Analyze fetch strategies** — for each relationship type, determine how many use explicit vs. implicit (JPA default) fetch behaviour.

   JPA fetch defaults (must be stated in the report for each relationship type found):
   - `@OneToMany` → default **LAZY**
   - `@ManyToMany` → default **LAZY**
   - `@ManyToOne` → default **EAGER**
   - `@OneToOne` → default **EAGER** (owning side)
   - `@ElementCollection` → default **EAGER**

   Run the following queries in parallel (read `metadata.total_items` only — these filters are reliable since `LAZY`/`EAGER` are unique to `FetchType`):
   - Explicit LAZY @OneToMany: `filters="annotation:contains:OneToMany,annotation:contains:LAZY"`
   - Explicit EAGER @OneToMany: `filters="annotation:contains:OneToMany,annotation:contains:EAGER"`
   - Explicit LAZY @ManyToOne: `filters="annotation:contains:ManyToOne,annotation:contains:LAZY"`
   - Explicit EAGER @ManyToOne: `filters="annotation:contains:ManyToOne,annotation:contains:EAGER"`
   - Explicit LAZY @OneToOne: `filters="annotation:contains:OneToOne,annotation:contains:LAZY"`
   - Explicit EAGER @OneToOne: `filters="annotation:contains:OneToOne,annotation:contains:EAGER"`
   - Explicit LAZY @ManyToMany: `filters="annotation:contains:ManyToMany,annotation:contains:LAZY"`
   - Explicit EAGER @ManyToMany: `filters="annotation:contains:ManyToMany,annotation:contains:EAGER"`
   - Explicit LAZY @ElementCollection: `filters="annotation:contains:ElementCollection,annotation:contains:LAZY"`
   - Explicit EAGER @ElementCollection: `filters="annotation:contains:ElementCollection,annotation:contains:EAGER"`

   Derive: **implicit (JPA default)** = total − explicit LAZY − explicit EAGER for each mapping type.

   > **Pagination:** Results are capped at 100 items per page. If the full-content fetch for any relationship type (needed in step 5 for listing violations) has `metadata.has_next: true`, fetch additional pages with `page=2`, `page=3`, etc. and concatenate all items before analysis. Always report the total item count from `metadata.total_items` alongside the number of items actually analyzed.

6. **Analyze join fetch mechanisms** — run the following in parallel (read `metadata.total_items` only):

   - `@Fetch(FetchMode.JOIN)` (Hibernate global join): `filters="annotation:contains:FetchMode.JOIN"`
   - `@Fetch(FetchMode.SUBSELECT)` (Hibernate subselect): `filters="annotation:contains:FetchMode.SUBSELECT"`
   - `@EntityGraph` (JPA standard, query-level): `filters="annotation:contains:EntityGraph"`
   - `@NamedEntityGraph` (JPA standard, class-level): `filters="annotation:contains:NamedEntityGraph,type:contains:class"`
   - `@BatchSize` (Hibernate batch loading): `filters="annotation:contains:BatchSize"`

7. **Identify Lombok @Data on entities** — fetch the full content of:
   `filters="annotation:contains:Data,annotation:contains:Entity,type:contains:class"`

   Then **parse the result file** using the MCP file parsing script (see appendix) and filter programmatically for items where:
   - the `annotations` array contains exactly `"@Data"` (string equality), AND
   - the `annotations` array contains a string starting with `"@Entity"`

   **Do not rely on the filter count alone.** `annotation:contains:Data` is a substring match that also hits annotations like `@Service("initializationDatabase")` (contains "Data" in "Database") and `@SpringBootApplication(exclude={DataSourceAutoConfiguration.class})`. Always verify by inspecting the parsed annotation strings.

   For each confirmed result record the class name and file path (translated per step 2).

8. **Analyze @ManyToMany anti-patterns** — fetch the full content of all `@ManyToMany` fields:
   `filters="annotation:contains:ManyToMany,type:contains:field"`

   Parse the result file using the MCP file parsing script (see appendix). For each confirmed `@ManyToMany` field, read the source file (translated per step 2) and check for the following 9 anti-patterns:

   **8a. `mappedBy` correctness:**
   - For each bidirectional `@ManyToMany` pair, verify that exactly one side declares `mappedBy` and the other declares `@JoinTable` (or uses the default).
   - Flag: **missing `mappedBy`** (both sides own the relationship → two separate join tables created silently), **wrong `mappedBy` value** (points to a non-existent or mismatched field), **both sides declare `mappedBy`** (no owning side → Hibernate ignores writes from both sides).

   **8b. `Set` vs `List` collection type:**
   - Read the source file and check the Java collection type of the `@ManyToMany` field.
   - Flag any field declared as `List` (or `Collection` without `Set` semantics).
   - **Why:** With `List`, Hibernate cannot compute a diff when the collection is modified. Instead, it **deletes all rows** from the join table for that entity and **re-inserts** the remaining ones. For a collection of 1000 items, removing 1 item generates 1 DELETE (all rows) + 999 INSERTs. With `Set` + proper `equals()`/`hashCode()`, Hibernate issues a single targeted DELETE.

   **8c. `CascadeType.REMOVE` or `CascadeType.ALL`:**
   - Check the `cascade` attribute in the annotation string.
   - Flag `CascadeType.REMOVE` or `CascadeType.ALL` (which includes REMOVE).
   - **Why:** On a `@ManyToMany`, cascade REMOVE does not just remove the join table row — it **deletes the target entity itself** from the database. If a `Student` has `@ManyToMany(cascade = CascadeType.REMOVE) Set<Course> courses`, removing a course from the student's collection deletes the `Course` entity, affecting all other students enrolled in it.

   **8d. `orphanRemoval = true`:**
   - Check for `orphanRemoval = true` in the annotation string.
   - **Why:** Same danger as `CascadeType.REMOVE`. When an element is removed from the collection, Hibernate treats it as an "orphan" and deletes the target entity from the database — not just the join table row.

   **8e. EAGER fetch on `@ManyToMany`:**
   - Already detected in step 5 (explicit EAGER count for `@ManyToMany`). Cross-reference those violations here.
   - Flag any `@ManyToMany` with `fetch = FetchType.EAGER` or `@ManyToMany` combined with `@Fetch(FetchMode.JOIN)`.
   - **Why:** `@ManyToMany` defaults to LAZY, which is correct. Overriding to EAGER loads the entire associated collection on every entity load, causing N+1 queries or cartesian products when multiple EAGER `@ManyToMany` collections are present on the same entity.

   **8f. Unidirectional `@ManyToMany` with `List`:**
   - Identify `@ManyToMany` fields that have no `mappedBy` and no corresponding inverse side on the target entity (unidirectional).
   - If the field uses `List`, flag it as a worst-case anti-pattern.
   - **Why:** Unidirectional `@ManyToMany` with `List` generates the most inefficient SQL of any JPA mapping. Hibernate cannot use the join table efficiently — element additions and removals trigger full table re-writes, and the lack of an inverse side prevents Hibernate from optimizing the join table operations.

   **8g. Missing `@JoinTable` specification:**
   - Check if the owning side of the `@ManyToMany` explicitly declares `@JoinTable`.
   - Flag if missing. This is not a performance issue but a naming convention concern.
   - **Why:** Without explicit `@JoinTable`, Hibernate auto-generates the join table name from the two entity names. This can produce unexpected or inconsistent table names (e.g., `student_courses` vs `course_students` depending on which entity is scanned first), making database administration harder.

   **8h. Missing `equals()`/`hashCode()` on collected entities:**
   - For each target entity in a `@ManyToMany(Set)` mapping, check whether the entity class has custom `equals()` and `hashCode()` methods (via source file read or Lombok `@EqualsAndHashCode`).
   - Flag entities used in `Set`-based `@ManyToMany` collections that rely on the default `Object.equals()`/`hashCode()`.
   - **Why:** `Set` semantics depend on `equals()`/`hashCode()`. Without proper implementations, Hibernate falls back to object identity, which breaks across sessions — detached entities re-read from the database will not match existing set members, causing duplicates or missed removals. This negates the performance benefit of using `Set` over `List`.

   **8i. Candidates for asymmetric refactoring:**
   - For each `@ManyToMany` mapping, recommend replacing it with two `@OneToMany` relationships pointing to an explicit join entity that maps the join table.
   - Prioritize candidates where any of the above anti-patterns (8a–8h) are present, or where the join table is likely to need extra attributes (audit columns, sort order, status, etc.).
   - **Why:** An explicit join entity (`A @OneToMany → AB @ManyToOne → B`) gives full control: its own `@Id` and lifecycle, support for extra columns (`created_at`, `role`, `quantity`), JPA callbacks (`@PrePersist`, `@PreRemove`), auditing, soft-delete, and efficient single-row INSERT/DELETE instead of delete-all + re-insert.

   > **Source reading strategy:** For checks 8b, 8c, 8d, 8f, 8g, and 8h, batch source file reads. Group `@ManyToMany` fields by their parent entity file, read each file once, and extract all needed information (collection type, cascade, orphanRemoval, `@JoinTable` presence, `equals()`/`hashCode()` methods) in a single pass.

9. **Analyze @ElementCollection anti-patterns** — fetch the full content of all `@ElementCollection` fields:
   `filters="annotation:contains:ElementCollection,type:contains:field"`

   Parse the result file using the MCP file parsing script (see appendix) and post-filter for items where the `annotations` array contains exactly `"@ElementCollection"` or a string starting with `"@ElementCollection("`. For each confirmed field, read the source file and check:

   **9a. Implicit EAGER fetch (default):**
   - Already detected in step 5. Cross-reference those violations here.
   - Flag any `@ElementCollection` without explicit `fetch = FetchType.LAZY`.
   - **Why:** Unlike `@OneToMany` and `@ManyToMany` which default to LAZY, `@ElementCollection` defaults to **EAGER**. This means the entire collection is loaded on every entity load, even when not needed. This is the same problem as implicit EAGER on `@ManyToOne`/`@OneToOne`, but less well-known because developers often assume collection mappings default to LAZY.

   **9b. Delete-all + re-insert behavior:**
   - Flag all `@ElementCollection` fields.
   - **Why:** Since collection elements (value types / embeddables) have no `@Id`, Hibernate cannot target individual rows for update or delete. When the collection is modified, Hibernate **deletes all rows** for that parent entity and **re-inserts** the remaining ones — the same behavior as `@ManyToMany` with `List`. This happens regardless of `Set` vs `List`, because the lack of identity is inherent to value types. For large collections, this generates massive write amplification.

   **9c. `Set` vs `List` / `equals()`/`hashCode()` dependency:**
   - Read the source file and check the Java collection type.
   - If using `Set`, check whether the embeddable class (if applicable) has proper `equals()`/`hashCode()`.
   - Flag `List` usage and missing `equals()`/`hashCode()` on embeddables.
   - **Why:** With `Set` and proper `equals()`/`hashCode()`, Hibernate can at least compute a diff and generate targeted DELETEs using the composite of all columns. With `List` or broken identity, it always falls back to the full delete + re-insert. For simple types (`String`, `Integer`), `Set` works out of the box; for `@Embeddable` types, custom `equals()`/`hashCode()` is required.

   **9d. Candidates for promotion to entity with `@OneToMany`:**
   - For each `@ElementCollection`, recommend evaluating whether it should be promoted to a full `@Entity` with an `@OneToMany` relationship.
   - Prioritize candidates where the collection is large, frequently modified, or where the embeddable already has multiple fields.
   - **Why:** A proper entity with its own `@Id` eliminates the delete-all + re-insert problem entirely. Hibernate can perform surgical single-row INSERTs, UPDATEs, and DELETEs. The entity can also be queried independently, cached, and managed with its own lifecycle.

10. **Analyze @MapsId misconfigurations** — fetch the full content of:
   `filters="annotation:contains:MapsId"`

   Parse the result file and post-filter for items where the `annotations` array contains exactly `"@MapsId"` or a string starting with `"@MapsId("`. For each confirmed field, read the source file and the parent entity class to check:

   **10a. `@MapsId` without matching ID attribute:**
   - If `@MapsId("someField")` is used, verify that the entity's `@EmbeddedId` class contains a field named `someField`. If `@MapsId` is used without a value, verify that the entity has an `@Id` field whose type matches the referenced entity's ID type.
   - **Why:** A mismatched `@MapsId` value causes a mapping exception at startup or, worse, silent data corruption if Hibernate falls back to a default mapping. The whole point of `@MapsId` is to share the primary key — if the mapping is wrong, the shared-key semantics are broken.

   **10b. `@MapsId` on `@ManyToMany`:**
   - Flag any `@MapsId` annotation on a field that also has `@ManyToMany`.
   - **Why:** `@MapsId` is only valid on `@ManyToOne` and `@OneToOne` associations. It maps the association's FK as the entity's PK (shared primary key pattern). `@ManyToMany` uses a join table, not a direct FK, so `@MapsId` has no meaning and will cause a mapping error or undefined behavior.

   **10c. `@MapsId` with `@GeneratedValue` conflict:**
   - Check whether the same entity has both `@MapsId` on an association and `@GeneratedValue` on its `@Id` field.
   - **Why:** `@MapsId` means "derive my ID from the parent association's key." `@GeneratedValue` means "auto-generate my ID." These are contradictory strategies — the ID should come from the parent, not from a sequence or auto-increment. This causes unpredictable behavior: Hibernate may ignore the generated value, or it may generate a value that conflicts with the parent's key.

   **10d. Missing `@MapsId` on shared-PK `@OneToOne`:**
   - Identify `@OneToOne` associations where the child entity's PK column is the same as its FK column (shared primary key pattern) but `@MapsId` is not declared.
   - This requires reading the source and checking for `@PrimaryKeyJoinColumn` or `@JoinColumn` pointing to the PK column.
   - **Why:** Without `@MapsId`, the child entity has a redundant PK column separate from the FK, wasting a column and an index. `@MapsId` tells Hibernate to reuse the FK as the PK, which is the correct mapping for a shared-PK `@OneToOne`. The absence of `@MapsId` is not an error but an optimization missed — an extra column, an extra unique constraint, and an extra index that serve no purpose.

11. **Analyze @OneToMany anti-patterns** — fetch the full content of all `@OneToMany` fields:
   `filters="annotation:contains:OneToMany,type:contains:field"`

   Parse the result file using the MCP file parsing script (see appendix). For each confirmed `@OneToMany` field, read the source file (translated per step 2) and check for the following 7 anti-patterns:

   **11a. Missing or wrong `mappedBy`:**
   - For each bidirectional `@OneToMany` / `@ManyToOne` pair, verify that the `@OneToMany` side declares `mappedBy` pointing to the correct field on the child entity.
   - Flag: **missing `mappedBy`** (the `@OneToMany` side owns the relationship, which is unusual and inefficient), **wrong `mappedBy` value** (points to a non-existent or mismatched field on the child), **both sides declare `mappedBy`** (no owning side).
   - **Why:** A `@OneToMany` without `mappedBy` is treated as the owning side. If no `@JoinColumn` is specified either, Hibernate creates a **separate join table** instead of using the FK column on the child table — a hidden extra table that is unnecessary and hurts insert/update performance with additional SQL statements.

   **11b. Unidirectional `@OneToMany` without `@JoinColumn`:**
   - Identify `@OneToMany` fields that have no `mappedBy` and no `@JoinColumn` annotation.
   - **Why:** Without `@JoinColumn`, Hibernate generates a **join table** (like `@ManyToMany`) instead of using a FK column on the child table. This produces 3 SQL statements per insert (insert child, insert join row, update child) instead of 1. Adding `@JoinColumn` on the `@OneToMany` tells Hibernate to use the child table's FK column directly.

   **11c. `CascadeType.REMOVE` or `CascadeType.ALL` risks:**
   - Check the `cascade` attribute in the annotation string.
   - Flag `CascadeType.REMOVE` or `CascadeType.ALL` on `@OneToMany` collections with potentially large child sets.
   - **Why:** `CascadeType.REMOVE` on `@OneToMany` deletes child entities one by one (N individual DELETE statements), not as a batch. For a parent with 1000 children, this generates 1000 DELETE queries instead of a single `DELETE FROM child WHERE parent_id = ?`. For large collections, this is a severe performance problem. Consider using bulk `@Query` DELETE or `@OnDelete(action = OnDeleteAction.CASCADE)` at the database level.

   **11d. `orphanRemoval` without `CascadeType.PERSIST` or `CascadeType.ALL`:**
   - Check if `orphanRemoval = true` is declared without `cascade` including `PERSIST` or `ALL`.
   - **Why:** `orphanRemoval` removes children when they are detached from the collection. But if new children added to the collection are not cascaded with PERSIST, they will not be saved — leading to transient entity exceptions at flush time. `orphanRemoval` and `CascadeType.PERSIST` must go together.

   **11e. `List` without `@OrderColumn` (bag semantics):**
   - Read the source file and check the Java collection type.
   - If the field is declared as `List` and has no `@OrderColumn` annotation, flag it.
   - **Why:** A `List` without `@OrderColumn` is treated as a **bag** by Hibernate — an unordered collection that allows duplicates. Bags have the same delete-all + re-insert behavior as `@ManyToMany` with `List`: Hibernate cannot compute targeted removals because there is no positional index, so it deletes all child FK references and re-inserts the remaining ones. Adding `@OrderColumn` gives the list a persistent index column, enabling efficient positional operations. Alternatively, use `Set` if ordering is not needed.

   **11f. Missing `equals()`/`hashCode()` on child entity:**
   - For each child entity type in a `@OneToMany(Set)` mapping, check whether the entity has custom `equals()` and `hashCode()`.
   - **Why:** Same as for `@ManyToMany` — `Set` semantics depend on `equals()`/`hashCode()`. Without proper implementations, detached and re-read entities will not match, causing duplicates in the set or missed removals. This negates the performance benefit of using `Set`.

   **11g. EAGER fetch override on `@OneToMany`:**
   - Already detected in step 5 (explicit EAGER count for `@OneToMany`). Cross-reference those violations here.
   - Flag any `@OneToMany` with `fetch = FetchType.EAGER`.
   - **Why:** `@OneToMany` defaults to LAZY, which is correct. Overriding to EAGER loads the entire child collection on every parent load. When multiple EAGER `@OneToMany` exist on the same entity, Hibernate produces a **cartesian product** — the result set size is the product of all collection sizes. Even a single EAGER `@OneToMany` can load thousands of child rows unconditionally.

   > **Source reading strategy:** Group fields by parent entity file and read each file once to extract all needed information (mappedBy, @JoinColumn presence, cascade, orphanRemoval, collection type, @OrderColumn, equals/hashCode) in a single pass.

12. **Analyze @ManyToOne anti-patterns** — fetch the full content of all `@ManyToOne` fields:
   `filters="annotation:contains:ManyToOne,type:contains:field"`

   Parse the result file using the MCP file parsing script (see appendix). For each confirmed `@ManyToOne` field, read the source file (translated per step 2) and check for the following 5 anti-patterns:

   **12a. Implicit EAGER fetch (default):**
   - Already detected in step 5 (implicit EAGER count for `@ManyToOne`). Cross-reference those violations here.
   - Flag any `@ManyToOne` without explicit `fetch = FetchType.LAZY`.
   - **Why:** `@ManyToOne` defaults to **EAGER**. Every time the child entity is loaded, Hibernate immediately loads the parent entity too — even when it's not needed. In a typical entity graph with multiple `@ManyToOne` associations (e.g., `Order → Customer`, `Order → Product`, `Order → Warehouse`), loading a single `Order` triggers 3 additional SELECT or JOIN queries. This is the single most common source of N+1 performance problems in JPA applications.

   **12b. `CascadeType.REMOVE` or `CascadeType.ALL`:**
   - Check the `cascade` attribute in the annotation string.
   - Flag `CascadeType.REMOVE` or `CascadeType.ALL` on `@ManyToOne`.
   - **Why:** Cascading REMOVE from child to parent is almost always a bug. Deleting an `OrderItem` would cascade to delete the `Order`, which would cascade to delete all other `OrderItem` entities on that order — a chain-reaction deletion. `CascadeType.ALL` includes REMOVE and is equally dangerous on `@ManyToOne`.

   **12c. `optional = true` (default) defeating LAZY:**
   - Check if the `@ManyToOne` has explicit `optional = false`. If not, it defaults to `optional = true`.
   - Flag `@ManyToOne(fetch = FetchType.LAZY)` without `optional = false` where the FK column is known to be NOT NULL.
   - **Why:** When `optional = true` (default), Hibernate **cannot** use a proxy for lazy loading in all cases. It must query the database to determine whether the association is null or non-null before deciding to return null or a proxy. With `optional = false`, Hibernate knows the association always exists and can safely return a proxy without querying. Adding `optional = false` (when the FK is NOT NULL) makes LAZY actually work as intended.

   **12d. Missing `equals()`/`hashCode()` on referenced entity:**
   - For entities referenced by `@ManyToOne` and used in `Set`-based collections on the inverse `@OneToMany` side, check for custom `equals()`/`hashCode()`.
   - **Why:** If the parent entity is referenced in child collections using `Set`, the parent needs proper `equals()`/`hashCode()` for set operations. Without them, Hibernate falls back to object identity, which breaks detach/reattach cycles.

   **12e. Explicit EAGER fetch override:**
   - Flag any `@ManyToOne` with explicit `fetch = FetchType.EAGER` (redundant since it's the default, but signals intentional EAGER loading).
   - **Why:** While this is the same as the default, an explicit EAGER declaration signals that someone deliberately chose EAGER loading. This should be scrutinized — the developer may have been unaware of the performance implications, or there may be a legitimate reason that should be documented.

   > **Source reading strategy:** Group fields by entity file and read each file once to check all attributes in a single pass.

13. **Analyze @OneToOne anti-patterns** — fetch the full content of all `@OneToOne` fields:
   `filters="annotation:contains:OneToOne,type:contains:field"`

   Parse the result file using the MCP file parsing script (see appendix). For each confirmed `@OneToOne` field, read the source file (translated per step 2) and check for the following 7 anti-patterns:

   **13a. Missing or wrong `mappedBy`:**
   - For each bidirectional `@OneToOne` pair, verify that exactly one side declares `mappedBy`.
   - Flag: **missing `mappedBy`** (both sides own the relationship → two FK columns), **wrong `mappedBy` value**, **both sides declare `mappedBy`** (no owning side).
   - **Why:** Without correct `mappedBy`, Hibernate creates two FK columns (one in each table) instead of one. This wastes storage, creates redundant constraints, and can cause data inconsistency if only one side is updated.

   **13b. LAZY not working on non-owning (mappedBy) side:**
   - Identify the non-owning side of each bidirectional `@OneToOne` (the side that declares `mappedBy`).
   - Flag if `fetch = FetchType.LAZY` is set on the non-owning side.
   - **Why:** LAZY loading **does not work** on the non-owning (mappedBy) side of `@OneToOne`. Hibernate must query the database to determine whether the associated entity exists (since the FK is on the other table), and once it queries, it has already loaded the data. The `LAZY` annotation is silently ignored, giving a false sense of optimization. Solutions: (1) make the relationship unidirectional, (2) use `@MapsId` to share the PK, or (3) use bytecode enhancement with `@LazyToOne(LazyToOneOption.NO_PROXY)`.

   **13c. Implicit EAGER fetch (default):**
   - Already detected in step 5 (implicit EAGER count for `@OneToOne`). Cross-reference those violations here.
   - Flag any `@OneToOne` without explicit `fetch = FetchType.LAZY`.
   - **Why:** `@OneToOne` defaults to EAGER on the owning side. Every load of the entity triggers an additional SELECT or JOIN for the associated entity. Combined with 13b (LAZY not working on non-owning side), `@OneToOne` is often the most problematic relationship for performance — both sides tend to load eagerly regardless of configuration.

   **13d. `optional = true` (default) defeating LAZY:**
   - Same check as 12c for `@ManyToOne`. Check if the `@OneToOne` has explicit `optional = false`.
   - Flag `@OneToOne(fetch = FetchType.LAZY)` without `optional = false` where the FK column is NOT NULL.
   - **Why:** Same as `@ManyToOne` — when `optional = true`, Hibernate cannot safely proxy the association without first checking if it exists. `optional = false` tells Hibernate the association is guaranteed non-null, enabling true lazy proxy loading.

   **13e. `CascadeType.REMOVE` or `CascadeType.ALL` risks:**
   - Check the `cascade` attribute.
   - Flag and assess the risk of cascade REMOVE on `@OneToOne`.
   - **Why:** On `@OneToOne`, cascade REMOVE is more often intentional (deleting a `User` should delete their `UserProfile`). However, it should still be reviewed because cascade from the child side to the parent is dangerous (same chain-reaction as `@ManyToOne`). Flag specifically cases where cascade REMOVE flows from child → parent.

   **13f. `orphanRemoval` without `CascadeType.PERSIST`:**
   - Same check as 11d. Flag `orphanRemoval = true` without cascade PERSIST.
   - **Why:** If you set the `@OneToOne` association to null (orphan removal triggers), but don't cascade PERSIST for new assignments, replacing the association with a new entity will fail with a transient entity exception.

   **13g. Missing `equals()`/`hashCode()` on associated entity:**
   - Check whether both sides of the `@OneToOne` relationship have proper `equals()`/`hashCode()`.
   - **Why:** When `@OneToOne` entities are used in collections (e.g., added to a `Set` or used as map keys), missing `equals()`/`hashCode()` causes identity issues across sessions. While less common than for collection-based mappings, it's still a correctness issue that can manifest as duplicate entries or missed lookups.

   > **Source reading strategy:** Group fields by entity file and read each file once to check all attributes in a single pass.

14. **Cross-reference with XXL table violations** — query CAST Imaging quality insights to identify tables flagged for excessive size, then correlate with JPA entities that have performance-sensitive anti-patterns.

   **14a. Retrieve XXL table violations:**
   - Call `mcp__imaging__applications_quality_insights` with `insights="structural-flaws"` to get structural flaw categories.
   - Then call `mcp__imaging__quality_insights` for the application and look for rules related to table size (e.g., "Avoid XXL tables", "Avoid tables with too many columns", or similar size/volume rules).
   - Alternatively, call `mcp__imaging__advisors` with `focus="list"` to find advisors, then `focus="rules"` to find table-size rules, then `focus="violations"` to get the flagged tables.
   - Collect the list of table names flagged as XXL or oversized.

   **14b. Map tables to JPA entities:**
   - For each flagged table, identify the JPA entity that maps to it by matching the table name to:
     - `@Table(name = "...")` annotation on entity classes
     - Default table name (entity class name) if no `@Table` is specified
     - Join tables from `@ManyToMany` / `@JoinTable` annotations
     - Collection tables from `@ElementCollection` / `@CollectionTable` annotations
   - Use `mcp__imaging__application_database_explorer` with `name_filter` to confirm the table exists and get its column details.

   **14c. Escalate severity for anti-patterns on large tables:**
   - For each entity/table that appears in the XXL violations list, check if it is involved in any of the following anti-patterns from previous steps:
     - `@ManyToMany` with `List` (step 8b) — delete-all + re-insert on a large join table
     - `@ManyToMany` unidirectional with `List` (step 8f) — worst-case on a large table
     - `@ElementCollection` delete-all + re-insert (step 9b) — massive write amplification on a large collection table
     - `@OneToMany` with `List` without `@OrderColumn` (step 11e) — bag semantics on a large child table
     - `CascadeType.REMOVE` on `@OneToMany` (step 11c) — N individual DELETEs on a large child table
     - Any EAGER fetch loading a large table unconditionally
   - Mark these as **Critical** severity in the report (upgraded from High) with a note: "Table flagged as XXL by CAST quality rules — performance impact is amplified."

   > **Why this matters:** The delete-all + re-insert problem, cascade REMOVE one-by-one deletion, and unbounded EAGER loading are all proportional to table size. A `@ManyToMany` with `List` on a table with 50 rows is a minor concern; the same anti-pattern on a table with 500,000 rows is a production incident waiting to happen. Cross-referencing with XXL table violations transforms generic recommendations into prioritized, data-driven findings.

15. **Analyze general JPA mapping pitfalls** — these checks apply across entity classes regardless of relationship type. For each check, query Imaging and/or read source files as indicated.

   **15a. `@GeneratedValue(strategy = GenerationType.IDENTITY)` disabling batch inserts:**
   - Fetch `filters="annotation:contains:GeneratedValue,type:contains:field"` and parse the result. For each field, check the annotation string for `GenerationType.IDENTITY` or `strategy = IDENTITY`.
   - **Category:** Performance
   - **Why:** `IDENTITY` strategy relies on database auto-increment and requires Hibernate to execute each INSERT **immediately** to retrieve the generated ID via `Statement.getGeneratedKeys()`. This completely **disables JDBC batch inserts** — even with `hibernate.jdbc.batch_size` configured, Hibernate must flush after every single INSERT. With `SEQUENCE` strategy and proper `allocationSize` (e.g., 50), Hibernate can pre-allocate IDs in memory and batch multiple INSERTs into a single JDBC call. For bulk operations, the difference is orders of magnitude (1000 individual round-trips vs. 20 batched calls).

   **15b. `@Inheritance` strategy pitfalls:**
   - Fetch `filters="annotation:contains:Inheritance,type:contains:class"` and parse the result. For each class, determine the strategy from the annotation string (`SINGLE_TABLE`, `TABLE_PER_CLASS`, `JOINED`). If no `@Inheritance` annotation is present on a class hierarchy, the default is `SINGLE_TABLE`.
   - Also read the source to identify class hierarchies: check for `extends` on `@Entity` classes to find parent-child relationships.
   - **Category:** Performance / Maintainability
   - **Why each strategy has pitfalls:**
     - **`SINGLE_TABLE`** (default): All subtypes share one table. Every column specific to a subtype must be **nullable** (even if logically required), causing table bloat and loss of NOT NULL constraints. A table with 5 subtypes of 10 columns each has 50 columns, most of which are NULL for any given row. Discriminator column must be maintained correctly.
     - **`TABLE_PER_CLASS`**: Each concrete subtype gets its own table with all inherited columns duplicated. Polymorphic queries (e.g., `SELECT FROM Parent`) generate a **`UNION ALL`** across all subtype tables — extremely slow with many subtypes or large tables. No shared FK constraints possible.
     - **`JOINED`**: Each subtype gets a table with only its own columns, joined to the parent table via PK. Every query on a subtype requires a **JOIN** to the parent table (and grandparent, etc.). Deep hierarchies (3+ levels) produce multi-table JOINs on every single query. Best normalized, but worst query performance for deep hierarchies.
   - Flag: all `TABLE_PER_CLASS` usages (almost always problematic), `JOINED` with hierarchy depth > 2, `SINGLE_TABLE` with > 5 subtypes or > 30 nullable columns.

   **15c. Large `@Lob` fields on entity:**
   - Fetch `filters="annotation:contains:Lob,type:contains:field"` and parse the result. For each `@Lob` field, check whether the entity also has a `@OneToOne(fetch = LAZY)` to a separate LOB entity, or whether bytecode enhancement is configured.
   - **Category:** Performance
   - **Why:** `@Lob` fields (BLOB/CLOB) are loaded with every entity fetch by default — Hibernate does not support lazy loading of individual fields without bytecode enhancement. A `@Lob` storing a 5MB image or document is loaded every time the entity is read, even in list queries that don't need the content. **Recommendation:** Move the LOB field to a separate `@Entity` with a `@OneToOne(fetch = FetchType.LAZY)` relationship, so it's only loaded on demand. Alternatively, enable Hibernate bytecode enhancement with `@LazyGroup`.

   **15d. Missing `@DynamicUpdate` on frequently updated wide entities:**
   - Fetch `filters="annotation:contains:DynamicUpdate,type:contains:class"` to find entities that already use it.
   - Cross-reference with all `@Entity` classes. For entities with many columns (check via `mcp__imaging__application_database_explorer` or source reading), flag those without `@DynamicUpdate`.
   - **Category:** Performance
   - **Why:** By default, Hibernate generates UPDATE statements with **all columns**, even when only one field changed. On a table with 30+ columns, this means sending 29 unchanged values to the database on every update. `@DynamicUpdate` tells Hibernate to generate UPDATE statements that include **only the modified columns**. This reduces network bandwidth, enables more efficient database execution plans, and avoids unnecessary write-ahead log entries. The trade-off is that Hibernate must diff the entity state at flush time, which has a small CPU cost — but for wide tables, the network and I/O savings far outweigh it.

   **15e. Missing `@Immutable` on read-only entities:**
   - Fetch `filters="annotation:contains:Immutable,type:contains:class"` to find entities that already use it.
   - Identify candidate read-only entities: lookup tables, reference data, configuration tables. Heuristics: entity classes with no setter methods, or tables with names like `*_type`, `*_status`, `*_code`, `*_config`, `*_reference`, `*_lookup`.
   - **Category:** Performance
   - **Why:** Hibernate **dirty-checks every managed entity** at flush time — comparing current field values against a snapshot taken at load time. For entities that never change (reference/lookup data), this is wasted work. `@Immutable` tells Hibernate to skip dirty checking entirely for that entity, and also prevents accidental modifications (Hibernate throws an exception if you try to update an `@Immutable` entity). For applications loading many reference entities per request, this reduces flush overhead significantly.

   **15f. Missing `@Version` for optimistic locking:**
   - Fetch `filters="annotation:contains:Version,type:contains:field"` to count entities with optimistic locking.
   - Compare against the total `@Entity` count from step 3. Flag entity classes that have no `@Version` field and are not `@Immutable`.
   - **Category:** Correctness / Data Integrity
   - **Why:** Without `@Version`, Hibernate uses a **last-write-wins** strategy for concurrent modifications. If two users load the same entity, modify different fields, and save — the second save silently overwrites the first user's changes with no error or warning. `@Version` (typically a `Long` or `Timestamp` field) enables optimistic locking: Hibernate adds `WHERE version = ?` to UPDATE statements, and throws `OptimisticLockException` if the version has changed since the entity was loaded. This is the standard JPA mechanism for preventing lost updates in concurrent environments.

   **15g. Composite `@EmbeddedId` without proper `equals()`/`hashCode()`:**
   - Fetch `filters="annotation:contains:EmbeddedId,type:contains:field"` and parse the result. For each `@EmbeddedId` field, identify the embeddable class and read its source to check for `equals()` and `hashCode()` methods (or Lombok `@EqualsAndHashCode`).
   - **Category:** Correctness
   - **Why:** The `@EmbeddedId` class is used as the **primary key** and as the key in Hibernate's first-level cache (persistence context). Without proper `equals()`/`hashCode()`, two instances representing the same composite key will not be considered equal. This corrupts the first-level cache — `em.find()` may return null for an entity that is already loaded, or load duplicate instances. It also breaks `Set`-based collections and `Map` lookups. `@EmbeddedId` classes **must** implement `equals()` and `hashCode()` based on all key fields, and must also implement `Serializable`.

   **15g2. Primitive types in composite key fields (`@EmbeddedId` / `@IdClass`):**
   - Reuse the `@EmbeddedId` classes identified in step 15g. Additionally, fetch `filters="annotation:contains:IdClass,type:contains:class"` to find entities using `@IdClass`. For each `@IdClass` annotation, extract the referenced class name from the annotation value (e.g., `@IdClass(OrderItemPK.class)` → `OrderItemPK`).
   - For each composite key class (whether `@Embeddable` used as `@EmbeddedId` or `@IdClass`), read the source file and inspect all declared fields. Flag any field whose type is a Java primitive: `int`, `long`, `short`, `byte`, `float`, `double`, `char`, `boolean`.
   - **Category:** Correctness
   - **Why:** The same `null`-detection problem that affects simple `@Id` fields applies — and is arguably **worse** — inside composite keys. A primitive field in a composite key cannot be `null`, so Hibernate cannot distinguish "key component is zero/false" from "key component was never set". This causes:
     - **Transient vs detached confusion:** `EntityManager.merge()` and `persist()` rely on key nullity to decide whether to INSERT or UPDATE. A composite key with all-zero primitives looks "set" even when it was never assigned.
     - **Broken equality:** If `equals()`/`hashCode()` includes primitive fields, the default value `0` is a valid hash input — two uninitialized keys are "equal" even though they represent different (or no) entities.
     - **Silent data corruption:** In `@IdClass` scenarios, JPA maps entity fields to key class fields by name. A primitive `int orderId` defaults to `0`, which may match an actual database row — leading to accidental overwrites.
   - Record each violation as: `KeyClass.fieldName → primitiveType` with recommended replacement (e.g., `int` → `Integer`, `long` → `Long`).

   **15h. Missing bidirectional sync utility methods:**
   - For each bidirectional relationship pair (`@OneToMany`/`@ManyToOne`, `@OneToOne`, `@ManyToMany`), read the owning entity source and check for helper methods that synchronize both sides (e.g., `addChild()` / `removeChild()` that update both the parent's collection and the child's back-reference).
   - **Category:** Correctness / Maintainability
   - **Why:** In a bidirectional relationship, JPA only persists the state of the **owning side**. If you add a child to the parent's collection but don't set the child's `@ManyToOne` back-reference, the relationship **will not be persisted**. Conversely, if you set the child's back-reference but don't add to the parent's collection, the in-memory state is inconsistent until the next database read. Utility methods that synchronize both sides prevent these bugs:
   ```java
   // On Parent entity:
   public void addChild(Child child) {
       children.add(child);
       child.setParent(this);
   }
   public void removeChild(Child child) {
       children.remove(child);
       child.setParent(null);
   }
   ```
   Flag entities with bidirectional relationships that lack such methods. This is a code quality / correctness concern, not a performance issue.

   **15i. `@Column(length=255)` default on String fields:**
   - For a sample of `@Entity` classes, read the source and check String fields for `@Column(length = ...)` annotations.
   - Flag String fields that rely on the default `length = 255` without an explicit override.
   - **Category:** Maintainability / Database Design
   - **Why:** JPA defaults all String fields to `VARCHAR(255)`. This is rarely the right size: short codes (`country_code`) waste space at 255, while long text fields (`description`, `comments`) silently truncate at 255 characters with no application-level error. Explicit `@Column(length = ...)` serves as documentation, prevents truncation bugs, and enables the DBA to optimize storage. Flag as informational — not a performance issue, but a schema quality concern.

   **15j. `@SecondaryTable` adding unnecessary JOINs:**
   - Fetch `filters="annotation:contains:SecondaryTable,type:contains:class"` and parse the result.
   - **Category:** Performance
   - **Why:** `@SecondaryTable` splits one entity across two database tables. Every load of the entity requires a **JOIN** between the primary and secondary tables — even when only primary table columns are needed. This is equivalent to an unconditional EAGER JOIN. In most cases, the secondary table data is better modeled as a separate `@Entity` with `@OneToOne(fetch = FetchType.LAZY)`, so it's only loaded on demand. Flag all `@SecondaryTable` usages and recommend evaluating whether a `@OneToOne` split would be more efficient.

   **15k. Second-level cache `@Cache` misconfiguration:**
   - Fetch `filters="annotation:contains:Cache,type:contains:class"` (note: this may match other annotations containing "Cache" — post-filter for `@org.hibernate.annotations.Cache` or `@javax.persistence.Cacheable`).
   - Also fetch `filters="annotation:contains:Cacheable,type:contains:class"`.
   - **Category:** Performance
   - **Why misconfigurations matter:**
     - **Wrong `CacheConcurrencyStrategy`**: `READ_WRITE` on a table that is never modified wastes overhead on cache invalidation — use `READ_ONLY` instead. `NONSTRICT_READ_WRITE` on a frequently modified table risks serving stale data.
     - **Missing cache on hot lookup tables**: Entities loaded on every request (e.g., `Country`, `Currency`, `Status`) that are not cached cause repeated database hits for data that rarely or never changes.
     - **Caching entities with EAGER collections**: If an entity with an EAGER `@OneToMany` is cached, the entire collection is cached with it — this bloats the cache and may cause stale collection data.
   - Flag: `@Cache` with `READ_WRITE` on `@Immutable` entities, entities frequently loaded (heuristic: referenced by many `@ManyToOne` fields) without `@Cacheable`, and cached entities with EAGER collections.

   > **Source reading strategy:** Many of these checks (15a, 15b, 15c, 15g) can be done entirely from Imaging annotation queries. Others (15d, 15e, 15f, 15h, 15i) require reading source files — batch these reads by grouping fields per entity file.

16. **Pre-flight checklist — complete before writing the report:**

   Before producing any output, verify that you have a real value (not "N" or "?") for each item below. If a value is missing, run the corresponding step now.

   - [ ] Total @Entity classes — from step 3 (`annotation:contains:Entity` total_items)
   - [ ] Total @Id fields confirmed — from step 4 Python filter (exact `@Id` annotation match)
   - [ ] @Id Java types determined — from step 4 `source_file_details` sample
   - [ ] @OneToMany total — from step 3
   - [ ] @ManyToOne total — from step 3
   - [ ] @OneToOne total — from step 3
   - [ ] @ManyToMany total — from step 3
   - [ ] @ElementCollection total — from step 3
   - [ ] Explicit LAZY / EAGER counts per mapping type (including @ElementCollection) — from step 5
   - [ ] Implicit (JPA default) counts derived — from step 5 math
   - [ ] @Fetch(FetchMode.JOIN) count — from step 6
   - [ ] @Fetch(FetchMode.SUBSELECT) count — from step 6
   - [ ] @EntityGraph count — from step 6
   - [ ] @NamedEntityGraph count — from step 6
   - [ ] @BatchSize count — from step 6
   - [ ] Lombok @Data entities confirmed via Python filter — from step 7
   - [ ] @ManyToMany anti-pattern checks (8a–8i) completed — from step 8
   - [ ] @ElementCollection anti-pattern checks (9a–9d) completed — from step 9
   - [ ] @MapsId misconfiguration checks (10a–10d) completed — from step 10
   - [ ] @OneToMany anti-pattern checks (11a–11g) completed — from step 11
   - [ ] @ManyToOne anti-pattern checks (12a–12e) completed — from step 12
   - [ ] @OneToOne anti-pattern checks (13a–13g) completed — from step 13
   - [ ] XXL table violations retrieved and cross-referenced — from step 14
   - [ ] General JPA pitfall checks (15a–15k) completed — from step 15

   Every section in the report template below **must appear in the output**, even when the count is zero. Do not omit a section because it has no violations — write "No violations found — ✓" instead.

17. **Produce the report** in this structure:

```
## JPA Entity Overview

| Metric | Count |
|--------|-------|
| Total @Entity classes | N |
| Total @Id fields (confirmed via Python filter, step 4) | N |
| @OneToMany mappings | N |
| @ManyToOne mappings | N |
| @OneToOne mappings | N |
| @ManyToMany mappings | N |
| @ElementCollection mappings | N |
| Total relationship mappings | N |

## Entity ID Types

### Distribution
| ID Type | Count | Assessment |
|---------|-------|------------|
| Long (wrapper) | N | ✓ Recommended |
| Integer (wrapper) | N | Acceptable |
| UUID | N | Acceptable (good for distributed systems) |
| String | N | ⚠ Anti-pattern |
| int (primitive) | N | ⚠ Anti-pattern |
| long (primitive) | N | ⚠ Anti-pattern |
| short / float / other primitive | N | ⚠ Anti-pattern |
| Unresolved | N | — |

### Issues
[For each String or primitive ID: entity name, field name, recommended replacement]

> **Why primitive IDs are problematic:** JPA uses `null` to distinguish transient from persisted entities.
> Primitive types cannot be `null`, so JPA cannot reliably detect unsaved entities, leading to incorrect
> `merge()` vs `persist()` decisions and potential data duplication.
>
> **Why String IDs are problematic:** No auto-generation support without custom generators, inconsistent
> length enforcement, poor index performance, and no natural numeric ordering.

## Fetch Strategy Analysis

### JPA Default Fetch Behaviour (reference)
| Annotation | JPA Default | Risk | Recommended Override |
|-----------|------------|------|----------------------|
| @OneToMany | LAZY | Low | Keep LAZY ✓ |
| @ManyToMany | LAZY | Low | Keep LAZY ✓ |
| @ManyToOne | EAGER | High | Override to LAZY |
| @OneToOne | EAGER | High | Override to LAZY |
| @ElementCollection | EAGER | High | Override to LAZY |

### Fetch Strategy Breakdown
| Mapping | Total | Explicit LAZY | Explicit EAGER | Implicit (JPA default) |
|---------|-------|--------------|----------------|------------------------|
| @OneToMany | N | N | N | N (default: LAZY) |
| @ManyToOne | N | N | N | N (default: EAGER ⚠) |
| @OneToOne | N | N | N | N (default: EAGER ⚠) |
| @ManyToMany | N | N | N | N (default: LAZY) |
| @ElementCollection | N | N | N | N (default: EAGER ⚠) |

### EAGER Violations

[For explicit EAGER (@OneToMany, @ManyToMany, @ManyToOne, @OneToOne): always list every violation — these are
deliberate overrides and should be few. For each: entity class, field name, annotation, recommendation.]

[For implicit EAGER (@ManyToOne, @OneToOne, and @ElementCollection with no fetch override — the dangerous default):
- If count ≤ 20: list all in a table (entity, field, annotation, recommendation).
- If count > 20: show the top 10 most impactful cases in the main table, then list the remainder
  in a collapsed <details> block. Prioritise: entities loaded on hot paths (e.g. Order, Product,
  Customer) over utility/description entities.]

> Unbounded EAGER loading is the most common cause of JPA performance problems. Every EAGER
> association fires an additional SQL JOIN or SELECT on every entity load, even when the association
> is not needed. Always override `@ManyToOne`, `@OneToOne`, and `@ElementCollection` to `fetch = FetchType.LAZY`
> and load associations on demand using `JOIN FETCH` in JPQL or `@EntityGraph` at the query level.

## Join Fetch Mechanisms

| Mechanism | Count | Scope | Notes |
|-----------|-------|-------|-------|
| @Fetch(FetchMode.JOIN) | N | Global (per field) | Hibernate-specific; fires JOIN on every load of the owning entity |
| @Fetch(FetchMode.SUBSELECT) | N | Global (per field) | Hibernate-specific; loads entire collection via subselect |
| @EntityGraph | N | Query-level | JPA standard; preferred — only activates when requested |
| @NamedEntityGraph | N | Class-level declaration | JPA standard; must be referenced by name in queries |
| @BatchSize | N | Global (per field) | Hibernate batching; reduces N+1 without full join |

[For @Fetch(FetchMode.JOIN): note that this is a static override — the JOIN fires unconditionally
on every load, equivalent to EAGER. Prefer @EntityGraph at the query level to retain control.]

## Lombok @Data on Entities

[If none found:]
No @Entity classes annotated with @Data detected — ✓

[If found:]
| Entity Class | File |
|-------------|------|
| EntityName | path/to/File.java |

**Why @Data is an anti-pattern on JPA entities:**
- `equals()` / `hashCode()` generated from **all fields** breaks entity identity semantics: two detached
  entities with the same DB row but different object states will not be equal, and entities stored in
  a `Set` may be "lost" after `merge()` changes their state.
- `toString()` traverses **all fields**, including lazy associations — this **will trigger lazy loading**,
  causing `LazyInitializationException` outside a transaction or N+1 queries inside one.
- `@Data` does not generate a no-arg constructor; JPA requires one. Unless `@NoArgsConstructor` is also
  present, entity instantiation by JPA will fail at runtime.

**Recommendation:** Replace `@Data` with:
- `@Getter` + `@Setter` for field access
- `@ToString(exclude = {"lazyField1", "lazyField2"})` to prevent lazy-load triggers
- Manual or `@EqualsAndHashCode` based solely on the business key or surrogate ID

## @ManyToMany Anti-Patterns

[If no @ManyToMany mappings found:]
No @ManyToMany mappings detected — ✓

[If found, produce a summary table first:]

| # | Anti-Pattern | Count | Severity |
|---|-------------|-------|----------|
| 1 | Missing or wrong `mappedBy` | N | ⚠ Critical — silent dual join tables or broken writes |
| 2 | `List` instead of `Set` | N | ⚠ High — delete-all + re-insert on every modification |
| 3 | `CascadeType.REMOVE` or `CascadeType.ALL` | N | ⚠ Critical — deletes target entity, not just association |
| 4 | `orphanRemoval = true` | N | ⚠ Critical — same as CascadeType.REMOVE |
| 5 | Explicit EAGER fetch | N | ⚠ High — N+1 queries or cartesian products |
| 6 | Unidirectional with `List` | N | ⚠ High — worst-case SQL generation |
| 7 | Missing `@JoinTable` specification | N | ℹ Low — naming convention concern |
| 8 | Missing `equals()`/`hashCode()` on target entity | N | ⚠ High — breaks Set semantics, negates Set benefit |
| 9 | Candidate for asymmetric refactoring | N | ℹ Recommendation |

[Then, for each anti-pattern with count > 0, list the violations in a detail table:]

### 8a. Missing or wrong `mappedBy`
| Entity Class | Field | Issue | File |
|-------------|-------|-------|------|
| ClassName | fieldName | Both sides own the relationship / mappedBy points to non-existent field / both sides declare mappedBy | path/to/File.java |

> **Why this matters:** Without a correct `mappedBy`, Hibernate creates **two separate join tables** (one per
> owning side). Changes made on one side are invisible to the other. This causes data duplication in the join
> tables and inconsistent reads depending on which side is queried.

### 8b. `List` instead of `Set`
| Entity Class | Field | Collection Type | File |
|-------------|-------|----------------|------|
| ClassName | fieldName | List / Collection | path/to/File.java |

> **Why this matters:** With `List`, Hibernate cannot compute a diff when the collection is modified. It
> **deletes all rows** from the join table for that entity and **re-inserts** the remaining ones. For a
> collection of 1000 items, removing 1 item generates 1 DELETE (all rows) + 999 INSERTs instead of a
> single targeted DELETE. Always use `Set` with proper `equals()`/`hashCode()`.

### 8c. `CascadeType.REMOVE` or `CascadeType.ALL`
| Entity Class | Field | Cascade Setting | File |
|-------------|-------|----------------|------|
| ClassName | fieldName | CascadeType.ALL / CascadeType.REMOVE | path/to/File.java |

> **Why this matters:** On a `@ManyToMany`, cascade REMOVE does not just remove the join table row — it
> **deletes the target entity itself** from the database. If `Student` has
> `@ManyToMany(cascade = CascadeType.REMOVE) Set<Course> courses`, removing a course from the student's
> collection deletes the `Course` entity entirely, affecting **all other students** enrolled in it.
> `CascadeType.ALL` includes REMOVE implicitly and is equally dangerous.

### 8d. `orphanRemoval = true`
| Entity Class | Field | File |
|-------------|-------|------|
| ClassName | fieldName | path/to/File.java |

> **Why this matters:** Same danger as `CascadeType.REMOVE`. When an element is removed from the collection,
> Hibernate treats it as an "orphan" and deletes the target entity from the database — not just the join
> table row. The target entity may still be referenced by other entities, causing constraint violations or
> silent data loss.

### 8e. Explicit EAGER fetch on `@ManyToMany`
| Entity Class | Field | File |
|-------------|-------|------|
| ClassName | fieldName | path/to/File.java |

> **Why this matters:** `@ManyToMany` defaults to LAZY, which is correct. Overriding to EAGER loads the
> entire associated collection on every entity load. When multiple EAGER `@ManyToMany` collections exist on
> the same entity, Hibernate produces a **cartesian product**, multiplying result rows exponentially. Even a
> single EAGER `@ManyToMany` fires an additional SQL JOIN or SELECT unconditionally.

### 8f. Unidirectional `@ManyToMany` with `List`
| Entity Class | Field | File |
|-------------|-------|------|
| ClassName | fieldName | path/to/File.java |

> **Why this matters:** This is the worst-case combination. Unidirectional `@ManyToMany` with `List`
> generates extremely inefficient SQL — Hibernate cannot optimize the join table operations without an
> inverse side, and the `List` type prevents diff computation. Every modification triggers a full join table
> rewrite with additional overhead from the unidirectional mapping.

### 8g. Missing `@JoinTable` specification
| Entity Class | Field | File |
|-------------|-------|------|
| ClassName | fieldName | path/to/File.java |

> **Note:** This is not a performance issue. Without explicit `@JoinTable`, Hibernate auto-generates the
> join table name from the two entity names. This can produce unexpected or inconsistent names, making
> database administration harder. Explicit `@JoinTable` improves clarity and predictability.

### 8h. Missing `equals()`/`hashCode()` on target entity
| Target Entity | Used in Set by | File |
|--------------|---------------|------|
| TargetEntity | OwnerEntity.fieldName | path/to/File.java |

> **Why this matters:** `Set` semantics depend on `equals()`/`hashCode()`. Without proper implementations,
> Hibernate falls back to object identity, which breaks across sessions — detached entities re-read from
> the database will not match existing set members, causing duplicates or missed removals. This **negates
> the performance benefit of using `Set` over `List`**, making the collection behave as poorly as a `List`
> despite the type declaration.

### 8i. Candidates for asymmetric refactoring
| Entity A | Entity B | Current Mapping | Anti-Patterns Found | Recommendation |
|----------|----------|----------------|--------------------|----|
| EntityA | EntityB | @ManyToMany | 8b, 8c (list violations) | Replace with EntityA @OneToMany → AB @ManyToOne → EntityB |

> **Recommended pattern:** Replace `A @ManyToMany ←→ B` with an explicit join entity:
> ```
> A @OneToMany → AB (join entity) @ManyToOne → B
> ```
> The join entity `AB` has its own `@Id`, lifecycle hooks, and support for extra columns (`created_at`,
> `role`, `quantity`, `sort_order`). Hibernate can perform surgical single-row INSERT/DELETE instead of
> delete-all + re-insert. Prioritize refactoring for mappings where multiple anti-patterns (8a–8h) are present.

## @ElementCollection Anti-Patterns

[If no @ElementCollection mappings found:]
No @ElementCollection mappings detected — ✓

[If found, produce a summary table first:]

| # | Anti-Pattern | Count | Severity |
|---|-------------|-------|----------|
| 1 | Implicit EAGER fetch (default) | N | ⚠ High — loaded on every entity access |
| 2 | Delete-all + re-insert behavior | N | ⚠ High — inherent to all @ElementCollection |
| 3 | `List` or missing `equals()`/`hashCode()` on embeddable | N | ⚠ High — prevents even partial diff optimization |
| 4 | Candidate for promotion to entity with @OneToMany | N | ℹ Recommendation |

### 9a. Implicit EAGER fetch
| Entity Class | Field | Explicit Fetch Override? | File |
|-------------|-------|------------------------|------|
| ClassName | fieldName | No (default: EAGER ⚠) / Yes (LAZY ✓) | path/to/File.java |

> **Why this matters:** Unlike `@OneToMany` and `@ManyToMany` which default to LAZY, `@ElementCollection`
> defaults to **EAGER**. The entire collection is loaded on every entity load, even when not needed. This is
> the same problem as implicit EAGER on `@ManyToOne`/`@OneToOne`, but less well-known because developers
> often assume all collection mappings default to LAZY. Always add `fetch = FetchType.LAZY`.

### 9b. Delete-all + re-insert behavior
[This applies to ALL @ElementCollection fields — list them all.]

| Entity Class | Field | Element Type | File |
|-------------|-------|-------------|------|
| ClassName | fieldName | String / EmbeddableName | path/to/File.java |

> **Why this matters:** Since collection elements (value types / embeddables) have **no `@Id`**, Hibernate
> cannot target individual rows for update or delete. When the collection is modified, Hibernate **deletes
> all rows** for that parent entity and **re-inserts** the remaining ones. This happens because without an
> identity column, Hibernate must match rows using the composite of all value columns — and for any
> modification, the safest strategy is a full rewrite. For large collections, this generates massive write
> amplification on every single-element change.

### 9c. `List` or missing `equals()`/`hashCode()` on embeddable
| Entity Class | Field | Collection Type | Embeddable has equals/hashCode? | File |
|-------------|-------|----------------|-------------------------------|------|
| ClassName | fieldName | List / Set | Yes / No / N/A (simple type) | path/to/File.java |

> **Why this matters:** With `Set` and proper `equals()`/`hashCode()`, Hibernate can at least compute a diff
> and generate targeted DELETEs using the composite of all columns, avoiding the full rewrite in some cases.
> With `List` or broken identity, it **always** falls back to delete-all + re-insert. For simple types
> (`String`, `Integer`), `Set` works out of the box; for `@Embeddable` types, custom `equals()`/`hashCode()`
> is required.

### 9d. Candidates for promotion to entity with @OneToMany
| Entity Class | Field | Element Type | Reason |
|-------------|-------|-------------|--------|
| ClassName | fieldName | EmbeddableName / String | Large collection / frequently modified / multiple fields |

> **Recommended pattern:** Promote the value type to a full `@Entity` with its own `@Id` and replace
> `@ElementCollection` with `@OneToMany`:
> ```
> Before: Parent @ElementCollection Set<Value> values
> After:  Parent @OneToMany Set<ValueEntity> values  (ValueEntity has @Id, @ManyToOne back-ref)
> ```
> A proper entity with its own `@Id` eliminates the delete-all + re-insert problem entirely. Hibernate
> can perform surgical single-row INSERTs, UPDATEs, and DELETEs. The entity can also be queried
> independently, cached, and managed with its own lifecycle.

## @MapsId Misconfigurations

[If no @MapsId annotations found:]
No @MapsId annotations detected — ✓ (check whether shared-PK @OneToOne patterns exist that should use @MapsId — see 10d.)

[If found, produce a summary table first:]

| # | Misconfiguration | Count | Severity |
|---|-----------------|-------|----------|
| 1 | `@MapsId` without matching ID attribute | N | ⚠ Critical — mapping exception or silent corruption |
| 2 | `@MapsId` on `@ManyToMany` | N | ⚠ Critical — invalid usage |
| 3 | `@MapsId` with `@GeneratedValue` conflict | N | ⚠ High — contradictory ID strategies |
| 4 | Missing `@MapsId` on shared-PK `@OneToOne` | N | ℹ Optimization opportunity |

### 10a. `@MapsId` without matching ID attribute
| Entity Class | Field | @MapsId Value | Expected ID Attribute | File |
|-------------|-------|--------------|----------------------|------|
| ClassName | fieldName | "someField" | EmbeddedId.someField not found | path/to/File.java |

> **Why this matters:** If `@MapsId("someField")` references a non-existent attribute in the `@EmbeddedId`
> class, or if `@MapsId` (without value) is used but the entity's `@Id` type does not match the referenced
> entity's ID type, Hibernate will throw a mapping exception at startup — or worse, silently map to the
> wrong column, causing data corruption.

### 10b. `@MapsId` on `@ManyToMany`
| Entity Class | Field | File |
|-------------|-------|------|
| ClassName | fieldName | path/to/File.java |

> **Why this matters:** `@MapsId` is only valid on `@ManyToOne` and `@OneToOne` associations. It maps the
> association's FK as the entity's PK (shared primary key pattern). `@ManyToMany` uses a join table, not a
> direct FK column, so `@MapsId` has no meaning and will cause a mapping error or undefined behavior.

### 10c. `@MapsId` with `@GeneratedValue` conflict
| Entity Class | @Id Field | @GeneratedValue Strategy | @MapsId on | File |
|-------------|-----------|------------------------|-----------|------|
| ClassName | id | IDENTITY / SEQUENCE / AUTO | associationField | path/to/File.java |

> **Why this matters:** `@MapsId` means "derive my ID from the parent association's key."
> `@GeneratedValue` means "auto-generate my ID." These are **contradictory strategies** — the ID should come
> from the parent, not from a sequence or auto-increment. Hibernate may ignore the generated value, generate
> a value that conflicts with the parent's key, or throw an exception depending on the JPA provider version.

### 10d. Missing `@MapsId` on shared-PK `@OneToOne`
| Child Entity | Parent Entity | Current Mapping | File |
|-------------|--------------|----------------|------|
| ChildEntity | ParentEntity | @OneToOne + @PrimaryKeyJoinColumn (no @MapsId) | path/to/File.java |

> **Why this matters:** Without `@MapsId`, the child entity has a redundant PK column separate from the FK,
> wasting a column and an index. `@MapsId` tells Hibernate to reuse the FK as the PK, which is the correct
> mapping for a shared-PK `@OneToOne`. The absence is not a runtime error but an optimization missed — an
> extra column, an extra unique constraint, and an extra index that serve no purpose.

## @OneToMany Anti-Patterns

[If no @OneToMany mappings found:]
No @OneToMany mappings detected — ✓

[If found, produce a summary table first:]

| # | Anti-Pattern | Count | Severity |
|---|-------------|-------|----------|
| 1 | Missing or wrong `mappedBy` | N | ⚠ High — hidden join table or broken writes |
| 2 | Unidirectional without `@JoinColumn` | N | ⚠ High — silent join table creation |
| 3 | `CascadeType.REMOVE` / `CascadeType.ALL` on large collections | N | ⚠ High — N individual DELETEs instead of bulk |
| 4 | `orphanRemoval` without `CascadeType.PERSIST` | N | ⚠ High — transient entity exceptions |
| 5 | `List` without `@OrderColumn` (bag semantics) | N | ⚠ High — delete-all + re-insert |
| 6 | Missing `equals()`/`hashCode()` on child entity | N | ⚠ High — breaks Set semantics |
| 7 | Explicit EAGER fetch override | N | ⚠ High — cartesian product risk |

[Then, for each anti-pattern with count > 0, list the violations in a detail table:]

### 11a. Missing or wrong `mappedBy`
| Entity Class | Field | Issue | File |
|-------------|-------|-------|------|
| ClassName | fieldName | No mappedBy (owning side) / wrong value / both sides declare mappedBy | path/to/File.java |

> **Why this matters:** A `@OneToMany` without `mappedBy` is treated as the owning side. If no `@JoinColumn`
> is specified either, Hibernate creates a **separate join table** instead of using the FK column on the child
> table — a hidden extra table that is unnecessary and hurts insert/update performance with additional SQL
> statements for every child operation.

### 11b. Unidirectional without `@JoinColumn`
| Entity Class | Field | Has @JoinColumn? | File |
|-------------|-------|-----------------|------|
| ClassName | fieldName | No | path/to/File.java |

> **Why this matters:** Without `@JoinColumn`, Hibernate generates a **join table** (like `@ManyToMany`)
> instead of using the FK column on the child table. This produces 3 SQL statements per child insert (insert
> child row, insert join table row, update child FK) instead of 1 simple INSERT. Adding `@JoinColumn` tells
> Hibernate to use the child table's FK column directly.

### 11c. `CascadeType.REMOVE` / `CascadeType.ALL` on large collections
| Entity Class | Field | Cascade Setting | File |
|-------------|-------|----------------|------|
| ClassName | fieldName | CascadeType.ALL / CascadeType.REMOVE | path/to/File.java |

> **Why this matters:** `CascadeType.REMOVE` on `@OneToMany` deletes child entities **one by one** (N
> individual DELETE statements), not as a batch. For a parent with 1000 children, this generates 1000 DELETE
> queries instead of a single `DELETE FROM child WHERE parent_id = ?`. For large collections, consider
> bulk `@Query` DELETE or `@OnDelete(action = OnDeleteAction.CASCADE)` at the database level instead.

### 11d. `orphanRemoval` without `CascadeType.PERSIST`
| Entity Class | Field | cascade value | orphanRemoval | File |
|-------------|-------|--------------|---------------|------|
| ClassName | fieldName | (missing PERSIST) | true | path/to/File.java |

> **Why this matters:** `orphanRemoval` removes children when they are detached from the collection. But if
> new children added to the collection are not cascaded with PERSIST, they will not be saved — leading to
> `TransientObjectException` at flush time. `orphanRemoval` and `CascadeType.PERSIST` must go together.

### 11e. `List` without `@OrderColumn` (bag semantics)
| Entity Class | Field | Collection Type | Has @OrderColumn? | File |
|-------------|-------|----------------|-------------------|------|
| ClassName | fieldName | List | No | path/to/File.java |

> **Why this matters:** A `List` without `@OrderColumn` is treated as a **bag** by Hibernate — an unordered
> collection that allows duplicates. Bags have the same delete-all + re-insert behavior as `@ManyToMany` with
> `List`: Hibernate cannot compute targeted removals because there is no positional index. Adding
> `@OrderColumn` gives the list a persistent index column, enabling efficient positional operations.
> Alternatively, use `Set` if ordering is not needed.

### 11f. Missing `equals()`/`hashCode()` on child entity
| Child Entity | Used in Set by | File |
|-------------|---------------|------|
| ChildEntity | ParentEntity.fieldName | path/to/File.java |

> **Why this matters:** Same as for `@ManyToMany` — `Set` semantics depend on `equals()`/`hashCode()`.
> Without proper implementations, detached and re-read entities will not match, causing duplicates in the
> set or missed removals. This negates the performance benefit of using `Set`.

### 11g. Explicit EAGER fetch override
| Entity Class | Field | File |
|-------------|-------|------|
| ClassName | fieldName | path/to/File.java |

> **Why this matters:** `@OneToMany` defaults to LAZY, which is correct. Overriding to EAGER loads the
> entire child collection on every parent load. When multiple EAGER `@OneToMany` exist on the same entity,
> Hibernate produces a **cartesian product** — the result set size is the product of all collection sizes.
> Even a single EAGER `@OneToMany` can load thousands of child rows unconditionally.

## @ManyToOne Anti-Patterns

[If no @ManyToOne mappings found:]
No @ManyToOne mappings detected — ✓

[If found, produce a summary table first:]

| # | Anti-Pattern | Count | Severity |
|---|-------------|-------|----------|
| 1 | Implicit EAGER fetch (default) | N | ⚠ High — most common source of N+1 |
| 2 | `CascadeType.REMOVE` / `CascadeType.ALL` | N | ⚠ Critical — chain-reaction deletion |
| 3 | `optional = true` (default) defeating LAZY | N | ⚠ Medium — LAZY may not take effect |
| 4 | Missing `equals()`/`hashCode()` on referenced entity | N | ⚠ High — breaks Set semantics on inverse side |
| 5 | Explicit EAGER fetch override | N | ℹ Low — same as default, but signals intentional choice |

[Then, for each anti-pattern with count > 0, list the violations in a detail table:]

### 12a. Implicit EAGER fetch (default)
| Entity Class | Field | Referenced Entity | File |
|-------------|-------|------------------|------|
| ClassName | fieldName | TargetEntity | path/to/File.java |

> **Why this matters:** `@ManyToOne` defaults to **EAGER**. Every time the child entity is loaded, Hibernate
> immediately loads the parent entity too — even when it's not needed. In a typical entity graph with multiple
> `@ManyToOne` associations (e.g., `Order → Customer`, `Order → Product`, `Order → Warehouse`), loading a
> single `Order` triggers 3 additional SELECT or JOIN queries. This is the **single most common source of
> N+1 performance problems** in JPA applications. Always add `fetch = FetchType.LAZY`.

### 12b. `CascadeType.REMOVE` / `CascadeType.ALL`
| Entity Class | Field | Cascade Setting | Referenced Entity | File |
|-------------|-------|----------------|------------------|------|
| ClassName | fieldName | CascadeType.ALL / CascadeType.REMOVE | TargetEntity | path/to/File.java |

> **Why this matters:** Cascading REMOVE from child to parent is almost always a bug. Deleting an `OrderItem`
> would cascade to delete the `Order`, which would cascade to delete **all other `OrderItem` entities** on
> that order — a chain-reaction deletion. `CascadeType.ALL` includes REMOVE and is equally dangerous on
> `@ManyToOne`. Cascade should flow from parent to child, not the reverse.

### 12c. `optional = true` (default) defeating LAZY
| Entity Class | Field | optional | fetch | FK nullable? | File |
|-------------|-------|---------|-------|-------------|------|
| ClassName | fieldName | true (default) | LAZY | No (NOT NULL) | path/to/File.java |

> **Why this matters:** When `optional = true` (default), Hibernate **cannot always** use a proxy for lazy
> loading. It may need to query the database to determine whether the association is null or non-null before
> deciding to return null or a proxy. With `optional = false`, Hibernate knows the association always exists
> and can safely return a proxy without querying. Adding `optional = false` (when the FK is NOT NULL) makes
> LAZY actually work as intended.

### 12d. Missing `equals()`/`hashCode()` on referenced entity
| Referenced Entity | Used by | File |
|------------------|---------|------|
| TargetEntity | ChildEntity.fieldName | path/to/File.java |

> **Why this matters:** If the parent entity is referenced in child collections using `Set` (on the inverse
> `@OneToMany` side), the parent needs proper `equals()`/`hashCode()` for set operations. Without them,
> Hibernate falls back to object identity, which breaks detach/reattach cycles.

### 12e. Explicit EAGER fetch override
| Entity Class | Field | File |
|-------------|-------|------|
| ClassName | fieldName | path/to/File.java |

> **Why this matters:** While EAGER is already the default for `@ManyToOne`, an explicit `fetch = FetchType.EAGER`
> declaration signals that someone deliberately chose EAGER loading. This should be scrutinized — the developer
> may have been unaware it's the default, or there may be a legitimate reason that should be documented.

## @OneToOne Anti-Patterns

[If no @OneToOne mappings found:]
No @OneToOne mappings detected — ✓

[If found, produce a summary table first:]

| # | Anti-Pattern | Count | Severity |
|---|-------------|-------|----------|
| 1 | Missing or wrong `mappedBy` | N | ⚠ High — dual FK columns |
| 2 | LAZY not working on non-owning (mappedBy) side | N | ⚠ High — LAZY silently ignored |
| 3 | Implicit EAGER fetch (default) | N | ⚠ High — additional SELECT on every load |
| 4 | `optional = true` (default) defeating LAZY | N | ⚠ Medium — LAZY may not take effect |
| 5 | `CascadeType.REMOVE` / `CascadeType.ALL` (child → parent direction) | N | ⚠ Critical — chain-reaction deletion |
| 6 | `orphanRemoval` without `CascadeType.PERSIST` | N | ⚠ High — transient entity exceptions |
| 7 | Missing `equals()`/`hashCode()` on associated entity | N | ⚠ Medium — identity issues across sessions |

[Then, for each anti-pattern with count > 0, list the violations in a detail table:]

### 13a. Missing or wrong `mappedBy`
| Entity Class | Field | Issue | File |
|-------------|-------|-------|------|
| ClassName | fieldName | Both sides own / wrong value / both declare mappedBy | path/to/File.java |

> **Why this matters:** Without correct `mappedBy`, Hibernate creates **two FK columns** (one in each table)
> instead of one. This wastes storage, creates redundant constraints, and can cause data inconsistency if only
> one side is updated.

### 13b. LAZY not working on non-owning (mappedBy) side
| Entity Class | Field | Side | fetch | File |
|-------------|-------|------|-------|------|
| ClassName | fieldName | Non-owning (mappedBy) | LAZY (ignored ⚠) | path/to/File.java |

> **Why this matters:** LAZY loading **does not work** on the non-owning (mappedBy) side of `@OneToOne`.
> Hibernate must query the database to determine whether the associated entity exists (since the FK is on the
> other table), and once it queries, it has already loaded the data. The `LAZY` annotation is **silently
> ignored**, giving a false sense of optimization. Solutions: (1) make the relationship unidirectional from
> the owning side only, (2) use `@MapsId` to share the PK (so Hibernate knows the association exists if the
> entity exists), or (3) use bytecode enhancement with `@LazyToOne(LazyToOneOption.NO_PROXY)`.

### 13c. Implicit EAGER fetch (default)
| Entity Class | Field | Side | File |
|-------------|-------|------|------|
| ClassName | fieldName | Owning / Non-owning | path/to/File.java |

> **Why this matters:** `@OneToOne` defaults to EAGER on the owning side. Every load of the entity triggers
> an additional SELECT or JOIN for the associated entity. Combined with 13b (LAZY not working on non-owning
> side), `@OneToOne` is often the **most problematic relationship for performance** — both sides tend to load
> eagerly regardless of configuration.

### 13d. `optional = true` (default) defeating LAZY
| Entity Class | Field | optional | fetch | FK nullable? | File |
|-------------|-------|---------|-------|-------------|------|
| ClassName | fieldName | true (default) | LAZY | No (NOT NULL) | path/to/File.java |

> **Why this matters:** Same as `@ManyToOne` — when `optional = true`, Hibernate cannot safely proxy the
> association without first checking if it exists. `optional = false` tells Hibernate the association is
> guaranteed non-null, enabling true lazy proxy loading on the owning side.

### 13e. `CascadeType.REMOVE` / `CascadeType.ALL` (child → parent direction)
| Entity Class | Field | Direction | Cascade Setting | File |
|-------------|-------|----------|----------------|------|
| ClassName | fieldName | child → parent | CascadeType.ALL / CascadeType.REMOVE | path/to/File.java |

> **Why this matters:** On `@OneToOne`, cascade REMOVE from parent → child is often intentional (deleting a
> `User` should delete their `UserProfile`). However, cascade from child → parent is dangerous — deleting a
> `UserProfile` would delete the `User`, which may cascade further. Flag specifically cases where cascade
> REMOVE flows in the child-to-parent direction.

### 13f. `orphanRemoval` without `CascadeType.PERSIST`
| Entity Class | Field | cascade value | orphanRemoval | File |
|-------------|-------|--------------|---------------|------|
| ClassName | fieldName | (missing PERSIST) | true | path/to/File.java |

> **Why this matters:** If you set the `@OneToOne` association to null (triggering orphan removal), but don't
> cascade PERSIST for new assignments, replacing the association with a new entity will fail with a
> `TransientObjectException` at flush time.

### 13g. Missing `equals()`/`hashCode()` on associated entity
| Associated Entity | Used by | File |
|------------------|---------|------|
| TargetEntity | OwnerEntity.fieldName | path/to/File.java |

> **Why this matters:** When `@OneToOne` entities are used in collections (e.g., added to a `Set` or used as
> map keys), missing `equals()`/`hashCode()` causes identity issues across sessions — detached entities
> re-read from the database will not be considered equal to their previous in-memory representation.

## XXL Table Cross-Reference

[If no XXL table violations found in CAST quality rules:]
No XXL table violations detected — table size data not available for severity escalation.

[If found:]

### Tables Flagged as XXL
| Table Name | Mapped Entity | CAST Rule | File |
|-----------|--------------|-----------|------|
| table_name | EntityClass | Rule name / ID | path/to/File.java |

### Escalated Anti-Patterns on Large Tables
| Entity / Table | Anti-Pattern | Original Severity | Escalated Severity | Impact |
|---------------|-------------|-------------------|-------------------|--------|
| EntityClass / table_name | @ManyToMany with List (8b) | ⚠ High | ⚠⚠ Critical | Delete-all + re-insert on XXL join table |
| EntityClass / table_name | CascadeType.REMOVE on @OneToMany (11c) | ⚠ High | ⚠⚠ Critical | N individual DELETEs on XXL child table |

> **Why this matters:** Performance anti-patterns are proportional to table size. A `@ManyToMany` with `List`
> on a table with 50 rows is a minor inefficiency; the same anti-pattern on a table with 500,000 rows
> generates 500,000 INSERT statements on every single-element removal. Cross-referencing with CAST XXL table
> violations transforms generic best-practice recommendations into **prioritized, data-driven findings** that
> highlight where the actual production risk lies.

## General JPA Mapping Pitfalls

| # | Pitfall | Category | Count | Severity |
|---|---------|----------|-------|----------|
| 15a | `@GeneratedValue(IDENTITY)` disabling batch inserts | Performance | N | ⚠ High — prevents JDBC batching |
| 15b | `@Inheritance` strategy issues | Performance / Maintainability | N | ⚠ Medium–High — depends on strategy |
| 15c | Large `@Lob` fields on entity | Performance | N | ⚠ High — loaded on every entity fetch |
| 15d | Missing `@DynamicUpdate` on wide entities | Performance | N | ℹ Medium — unnecessary column writes |
| 15e | Missing `@Immutable` on read-only entities | Performance | N | ℹ Medium — unnecessary dirty checking |
| 15f | Missing `@Version` for optimistic locking | Correctness / Data Integrity | N | ⚠ High — silent lost updates |
| 15g | `@EmbeddedId` without `equals()`/`hashCode()` | Correctness | N | ⚠ Critical — corrupts first-level cache |
| 15h | Missing bidirectional sync utility methods | Correctness / Maintainability | N | ℹ Medium — inconsistent in-memory state |
| 15i | `@Column(length=255)` default on String fields | Maintainability / Database Design | N | ℹ Low — schema quality concern |
| 15j | `@SecondaryTable` adding unnecessary JOINs | Performance | N | ⚠ Medium — unconditional JOIN per load |
| 15k | Second-level cache `@Cache` misconfiguration | Performance | N | ⚠ Medium — stale data or missed caching |

[For each pitfall with count > 0, list the violations in a detail table:]

### 15a. `@GeneratedValue(IDENTITY)` disabling batch inserts
| Entity Class | @Id Field | Strategy | File |
|-------------|-----------|----------|------|
| ClassName | id | IDENTITY | path/to/File.java |

> **Category: Performance.** `IDENTITY` relies on database auto-increment and forces Hibernate to execute
> each INSERT immediately to retrieve the generated ID. This **completely disables JDBC batch inserts** — even
> with `hibernate.jdbc.batch_size` configured. Switch to `SEQUENCE` with proper `allocationSize` (e.g., 50)
> to enable batched INSERTs.

### 15b. `@Inheritance` strategy issues
| Parent Entity | Strategy | Subtypes | Issue | File |
|--------------|----------|----------|-------|------|
| ParentClass | TABLE_PER_CLASS / JOINED / SINGLE_TABLE | N | UNION ALL queries / deep JOIN chain / excessive nullable columns | path/to/File.java |

> **Category: Performance / Maintainability.**
> - **`TABLE_PER_CLASS`**: Polymorphic queries generate `UNION ALL` across all subtype tables — almost always problematic.
> - **`JOINED`** with depth > 2: Every query requires multi-table JOINs up the hierarchy chain.
> - **`SINGLE_TABLE`** with many subtypes: Table bloat from nullable columns across all subtypes.

### 15c. Large `@Lob` fields on entity
| Entity Class | Field | Lob Type | Separate Entity? | File |
|-------------|-------|---------|-----------------|------|
| ClassName | fieldName | BLOB / CLOB | No ⚠ | path/to/File.java |

> **Category: Performance.** `@Lob` fields are loaded with every entity fetch by default. A 5MB image or
> document is loaded even in list queries. Move to a separate `@Entity` with `@OneToOne(fetch = LAZY)`, or
> enable Hibernate bytecode enhancement with `@LazyGroup`.

### 15d. Missing `@DynamicUpdate` on wide entities
| Entity Class | Column Count | Has @DynamicUpdate? | File |
|-------------|-------------|--------------------|----- |
| ClassName | N | No ⚠ | path/to/File.java |

> **Category: Performance.** Hibernate generates UPDATE with **all columns** by default. On tables with 30+
> columns, this sends 29 unchanged values on every update. `@DynamicUpdate` generates UPDATE with only
> modified columns, reducing network bandwidth and I/O.

### 15e. Missing `@Immutable` on read-only entities
| Entity Class | Candidate Reason | File |
|-------------|-----------------|------|
| ClassName | Lookup table / no setters / *_type naming | path/to/File.java |

> **Category: Performance.** Hibernate dirty-checks every managed entity at flush time. `@Immutable` skips
> dirty checking entirely and prevents accidental modifications. Apply to lookup, reference, and configuration
> entities that never change at runtime.

### 15f. Missing `@Version` for optimistic locking
| Entity Class | Has @Version? | Has @Immutable? | File |
|-------------|--------------|----------------|------|
| ClassName | No ⚠ | No | path/to/File.java |

> **Category: Correctness / Data Integrity.** Without `@Version`, concurrent modifications use last-write-wins
> — the second save silently overwrites the first user's changes. `@Version` enables optimistic locking:
> Hibernate adds `WHERE version = ?` to UPDATEs and throws `OptimisticLockException` on conflicts.

### 15g. `@EmbeddedId` without `equals()`/`hashCode()`
| Entity Class | EmbeddedId Class | Has equals/hashCode? | File |
|-------------|-----------------|---------------------|------|
| ClassName | KeyClass | No ⚠ | path/to/File.java |

> **Category: Correctness.** The `@EmbeddedId` class is used as the primary key and as the key in Hibernate's
> first-level cache. Without proper `equals()`/`hashCode()`, `em.find()` may return null for already-loaded
> entities, or load duplicates. `@EmbeddedId` classes **must** implement both methods and `Serializable`.

### 15g2. Primitive types in composite key fields
[If none:] No composite key classes use primitive types — ✓

[If found:]
| Key Class | Key Type | Field | Primitive Type | Recommended | File |
|-----------|----------|-------|---------------|-------------|------|
| OrderItemPK | @EmbeddedId | orderId | int | Integer | path/to/OrderItemPK.java |
| OrderItemPK | @EmbeddedId | productId | long | Long | path/to/OrderItemPK.java |
| InvoiceLinePK | @IdClass | invoiceId | int | Integer | path/to/InvoiceLinePK.java |

**Total: N primitive field(s) across N composite key class(es)**

> **Category: Correctness.** Primitive types in composite keys cannot be `null`, so Hibernate cannot distinguish
> "key component is zero" from "key component was never set". This breaks transient/detached detection
> (`merge()` and `persist()` rely on key nullity), corrupts `equals()`/`hashCode()` (uninitialized keys
> hash identically), and risks silent data overwrites when a default `0` matches an actual database row.
> All composite key fields should use wrapper types (`Integer`, `Long`, etc.).

### 15h. Missing bidirectional sync utility methods
| Entity Class | Relationship | Inverse Side | Has sync methods? | File |
|-------------|-------------|-------------|------------------|------|
| ParentClass | @OneToMany children | ChildClass.parent | No ⚠ | path/to/File.java |

> **Category: Correctness / Maintainability.** JPA only persists the owning side. Without `addChild()`/
> `removeChild()` methods that update both sides, the relationship may not be persisted or the in-memory
> state may be inconsistent until the next database read.

### 15i. `@Column(length=255)` default on String fields
| Entity Class | Field | Has explicit length? | File |
|-------------|-------|---------------------|------|
| ClassName | fieldName | No (default 255) | path/to/File.java |

> **Category: Maintainability / Database Design.** JPA defaults all String fields to `VARCHAR(255)`. Short
> codes waste space; long text fields silently truncate. Explicit `@Column(length = ...)` prevents bugs and
> enables storage optimization. Informational finding — not a performance issue.

### 15j. `@SecondaryTable` adding unnecessary JOINs
| Entity Class | Secondary Table | File |
|-------------|----------------|------|
| ClassName | secondary_table_name | path/to/File.java |

> **Category: Performance.** `@SecondaryTable` forces a JOIN between primary and secondary tables on every
> entity load — even when secondary table columns are not needed. Consider replacing with a separate `@Entity`
> and `@OneToOne(fetch = LAZY)` to load on demand.

### 15k. Second-level cache `@Cache` misconfiguration
| Entity Class | Cache Strategy | Issue | File |
|-------------|---------------|-------|------|
| ClassName | READ_WRITE / NONSTRICT_READ_WRITE / (missing) | Wrong strategy for @Immutable / missing on hot lookup / cached with EAGER collection | path/to/File.java |

> **Category: Performance.** Misconfigured caching causes either stale data (wrong strategy) or repeated
> database hits (missing cache on hot tables). Use `READ_ONLY` for `@Immutable` entities, add `@Cacheable`
> to frequently loaded lookup tables, and avoid caching entities with EAGER collections.

## Good Practices Observed
[Bullet list of what is done well — e.g. consistent use of Long wrapper IDs, all @ManyToOne
overridden to LAZY, use of @EntityGraph for query-level fetching, no @Data on entities, etc.
If a category has no violations, state "No violations found" rather than omitting it.]

## Summary
| Category | Count | Status |
|----------|-------|--------|
| Total @Entity classes | N | — |
| String IDs | N | ⚠ / ✓ |
| Primitive type IDs | N | ⚠ / ✓ |
| Explicit EAGER mappings (incl. @OneToMany, @ManyToMany overrides) | N | ⚠ / ✓ |
| Implicit EAGER (@ManyToOne / @OneToOne / @ElementCollection without override) | N | ⚠ / ✓ |
| @Fetch(FetchMode.JOIN) (global join fetch) | N | ⚠ / ✓ |
| @EntityGraph / @BatchSize in use (query-level fetch control) | N | ⚠ if 0 and EAGER violations exist |
| Entities using Lombok @Data | N | ⚠ / ✓ |
| @ManyToMany anti-patterns (total violations across 8a–8i) | N | ⚠ / ✓ |
| @ElementCollection anti-patterns (total violations across 9a–9d) | N | ⚠ / ✓ |
| @MapsId misconfigurations (total violations across 10a–10d) | N | ⚠ / ✓ |
| @OneToMany anti-patterns (total violations across 11a–11g) | N | ⚠ / ✓ |
| @ManyToOne anti-patterns (total violations across 12a–12e) | N | ⚠ / ✓ |
| @OneToOne anti-patterns (total violations across 13a–13g) | N | ⚠ / ✓ |
| XXL table escalations | N | ⚠⚠ Critical if anti-patterns on large tables / ✓ |
| @GeneratedValue(IDENTITY) disabling batch inserts (15a) | N | ⚠ / ✓ |
| @Inheritance strategy issues (15b) | N | ⚠ / ✓ |
| Large @Lob fields on entity (15c) | N | ⚠ / ✓ |
| Missing @DynamicUpdate on wide entities (15d) | N | ℹ / ✓ |
| Missing @Immutable on read-only entities (15e) | N | ℹ / ✓ |
| Missing @Version for optimistic locking (15f) | N | ⚠ / ✓ |
| @EmbeddedId without equals/hashCode (15g) | N | ⚠ / ✓ |
| Primitive types in composite key fields (15g2) | N | ⚠ / ✓ |
| Missing bidirectional sync methods (15h) | N | ℹ / ✓ |
| @Column(length=255) defaults (15i) | N | ℹ / ✓ |
| @SecondaryTable unnecessary JOINs (15j) | N | ⚠ / ✓ |
| @Cache misconfiguration (15k) | N | ⚠ / ✓ |
```

Every section of the report must appear, even when the finding count is zero — write "No violations found — ✓" rather than omitting the section.

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Severity |
|---|-------------|-------------------|---------|
| 1 | ID types (String / primitives) | N entity/entities | ⚠ Fix needed / ✓ None |
| 2 | Explicit EAGER fetch | N field(s) | ⚠ N+1 risk / ✓ None |
| 3 | Implicit EAGER fetch (@ManyToOne / @OneToOne / @ElementCollection without override) | N field(s) | ⚠ N+1 risk / ✓ None |
| 4 | @Fetch(FetchMode.JOIN) (Hibernate) | N field(s) | ⚠ Unconditional JOIN / ✓ None |
| 5 | No query-level fetch control | N entity/entities affected | ⚠ If EAGER violations / ✓ |
| 6 | Lombok @Data on JPA entities | N entity/entities | ⚠ Anti-pattern / ✓ None |
| 7 | @ManyToMany anti-patterns (mappedBy, List, cascade, orphanRemoval, EAGER, equals/hashCode, @JoinTable) | N violation(s) | ⚠ Performance / data integrity / ✓ None |
| 8 | @ElementCollection anti-patterns (implicit EAGER, delete-all + re-insert, List/equals) | N violation(s) | ⚠ Performance / ✓ None |
| 9 | @MapsId misconfigurations (missing match, @ManyToMany usage, @GeneratedValue conflict, missing on shared-PK) | N violation(s) | ⚠ Mapping error / ✓ None |
| 10 | @OneToMany anti-patterns (mappedBy, unidirectional join table, cascade, orphanRemoval, bag, equals/hashCode, EAGER) | N violation(s) | ⚠ Performance / data integrity / ✓ None |
| 11 | @ManyToOne anti-patterns (implicit EAGER, cascade REMOVE, optional=true, equals/hashCode) | N violation(s) | ⚠ Performance / data integrity / ✓ None |
| 12 | @OneToOne anti-patterns (mappedBy, LAZY on non-owning side, EAGER, optional=true, cascade, orphanRemoval, equals/hashCode) | N violation(s) | ⚠ Performance / correctness / ✓ None |
| 13 | XXL table cross-reference (severity escalation for anti-patterns on large tables) | N escalation(s) | ⚠⚠ Critical / ✓ None |
| 14 | @GeneratedValue(IDENTITY) disabling batch inserts | N entity/entities | ⚠ Performance / ✓ None |
| 15 | @Inheritance strategy issues (TABLE_PER_CLASS, deep JOINED, bloated SINGLE_TABLE) | N entity/entities | ⚠ Performance / ✓ None |
| 16 | Large @Lob fields loaded eagerly | N field(s) | ⚠ Performance / ✓ None |
| 17 | Missing @Version for optimistic locking | N entity/entities | ⚠ Correctness / ✓ None |
| 18 | @EmbeddedId without equals/hashCode | N class(es) | ⚠ Correctness / ✓ None |
| 19 | Primitive types in composite key fields (@EmbeddedId / @IdClass) | N field(s) | ⚠ Correctness / ✓ None |
| 20 | Missing @DynamicUpdate, @Immutable, @Cache, @SecondaryTable, sync methods, @Column length | N finding(s) | ℹ Various / ✓ None |

**By priority (most critical first):**

| Priority | Issue | Detail | Recommendation |
|----------|---------|--------|----------------|
| 1 | @ManyToMany CascadeType.REMOVE / ALL / orphanRemoval | N field(s) — deletes target entity, not just association | Remove cascade REMOVE/ALL and orphanRemoval on @ManyToMany |
| 2 | @ManyToOne CascadeType.REMOVE / ALL | N field(s) — chain-reaction deletion child → parent → siblings | Remove cascade REMOVE/ALL on @ManyToOne |
| 3 | @ManyToMany missing or wrong `mappedBy` | N field(s) — dual join tables or broken writes | Fix mappedBy to point to correct inverse field |
| 4 | @OneToOne LAZY on non-owning (mappedBy) side | N field(s) — LAZY silently ignored | Make unidirectional, use @MapsId, or bytecode enhancement |
| 5 | @Data on JPA entities | N entity/entities — hashCode/toString on lazy proxies | Replace with @Getter/@Setter + @EqualsAndHashCode on ID |
| 6 | @MapsId without matching ID / on @ManyToMany / with @GeneratedValue | N field(s) — mapping exception or silent corruption | Fix ID attribute mapping or remove conflicting annotations |
| 7 | String ID type | N entity/entities — no reliable auto-generation | Migrate to `Long` with `@GeneratedValue` |
| 8 | Explicit EAGER on @OneToMany / @ManyToMany | N field(s) — mass loading / cartesian products | Switch to LAZY + @EntityGraph at query level |
| 9 | Implicit EAGER (@ManyToOne / @OneToOne / @ElementCollection) | N field(s) — silent N+1 | Add `fetch = FetchType.LAZY` |
| 10 | @OneToMany unidirectional without @JoinColumn | N field(s) — hidden join table | Add @JoinColumn or make bidirectional with mappedBy |
| 11 | @ManyToMany with List instead of Set | N field(s) — delete-all + re-insert | Change to Set with proper equals()/hashCode() |
| 12 | @OneToMany List without @OrderColumn (bag) | N field(s) — delete-all + re-insert | Add @OrderColumn or use Set |
| 13 | @OneToMany CascadeType.REMOVE on large collections | N field(s) — N individual DELETEs | Use bulk @Query DELETE or DB-level cascade |
| 14 | `optional = true` defeating LAZY (@ManyToOne / @OneToOne) | N field(s) — LAZY may not take effect | Add `optional = false` where FK is NOT NULL |
| 15 | orphanRemoval without CascadeType.PERSIST (@OneToMany / @OneToOne) | N field(s) — transient entity exceptions | Add CascadeType.PERSIST alongside orphanRemoval |
| 16 | @ElementCollection delete-all + re-insert / List usage | N field(s) — write amplification | Promote to entity with @OneToMany or use Set |
| 17 | @Fetch(FetchMode.JOIN) | N field(s) — unconditional JOIN | Replace with @EntityGraph or @BatchSize |
| 18 | Missing equals()/hashCode() on entities used in Set collections | N entity/entities — broken Set semantics | Implement based on business key or surrogate ID |
| 19 | No @EntityGraph / @BatchSize | No query-level fetch control | Add @EntityGraph to the relevant repositories |
| 20 | @ManyToMany candidates for asymmetric refactoring | N mapping(s) | Replace with @OneToMany → join entity → @ManyToOne |
| 21 | @ElementCollection candidates for entity promotion | N field(s) | Promote to entity with @OneToMany |
| 22 | Missing @MapsId on shared-PK @OneToOne | N field(s) — redundant PK column | Add @MapsId to reuse FK as PK |
| 23 | @EmbeddedId without equals()/hashCode() | N class(es) — corrupts first-level cache | Implement equals/hashCode on all key fields + Serializable |
| 24 | Primitive types in composite key fields | N field(s) — broken null detection, silent overwrites | Replace primitives with wrapper types (int→Integer, long→Long) |
| 25 | Missing @Version for optimistic locking | N entity/entities — silent lost updates | Add @Version Long version field |
| 26 | @GeneratedValue(IDENTITY) disabling batch inserts | N entity/entities — no JDBC batching | Switch to SEQUENCE with allocationSize |
| 27 | Large @Lob fields on entity | N field(s) — loaded on every fetch | Move to separate @OneToOne(fetch=LAZY) entity |
| 28 | @Inheritance TABLE_PER_CLASS | N hierarchy/hierarchies — UNION ALL on polymorphic queries | Switch to JOINED or SINGLE_TABLE |
| 29 | @Inheritance JOINED with depth > 2 | N hierarchy/hierarchies — multi-table JOINs | Flatten hierarchy or use SINGLE_TABLE |
| 30 | @SecondaryTable unnecessary JOINs | N entity/entities — unconditional JOIN per load | Replace with @OneToOne(fetch=LAZY) |
| 31 | Missing @DynamicUpdate on wide entities | N entity/entities — full UPDATE on every change | Add @DynamicUpdate to entities with 30+ columns |
| 32 | Missing @Immutable on read-only entities | N entity/entities — unnecessary dirty checking | Add @Immutable to lookup/reference entities |
| 33 | @Cache misconfiguration | N entity/entities — stale data or missed caching | Match CacheConcurrencyStrategy to entity mutability |
| 34 | Missing bidirectional sync methods | N entity/entities — inconsistent in-memory state | Add addChild()/removeChild() utility methods |
| 35 | @Column(length=255) defaults | N field(s) — implicit VARCHAR(255) | Add explicit @Column(length=N) |

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

### Usage pattern

```python
items, meta = parse_mcp_result_file('/path/to/tool-result.txt')
print('Total items (all pages):', meta['total_items'])
print('Items in this file:', len(items))

# Example: filter for exact @Id annotation
id_fields = [
    item for item in items
    if any(ann.strip() == '@Id' or ann.strip().startswith('@Id(')
           for ann in item.get('annotations', []))
]

# Example: filter for exact @Data AND @Entity
data_entities = [
    item for item in items
    if any(ann.strip() == '@Data' for ann in item.get('annotations', []))
    and any('@Entity' in ann for ann in item.get('annotations', []))
]
```

### Key caveats

| Filter | Problem | Fix |
|--------|---------|-----|
| `annotation:contains:Id` | Matches `@Valid`, `@JoinColumn(name="..._ID")` — too broad | After fetching, filter with `ann.strip() == '@Id'` in Python |
| `annotation:contains:Data` | Matches `@Service("initializationDatabase")`, `exclude={DataSourceAutoConfiguration.class}` — too broad | After fetching, filter with `ann.strip() == '@Data'` in Python |
| `mangling:contains:String/Long/...` | `mangling` is always empty for Java fields | Use `mcp__imaging__source_file_details` and read getter return types instead |

## Optional — Create a Saved View

After producing the report, if any violations were found, offer to create a CAST Imaging saved view per finding group. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the violating objects?
- Entities with EAGER fetch violations: N fields/classes
- Entities with Lombok @Data: N classes
- Entities with String/primitive IDs: N classes
- @ManyToMany anti-pattern violations: N fields
- @ElementCollection anti-pattern violations: N fields
- @MapsId misconfigurations: N fields
- @OneToMany anti-pattern violations: N fields
- @ManyToOne anti-pattern violations: N fields
- @OneToOne anti-pattern violations: N fields
Type yes or specify which group(s) to include.
```

For each group the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="JPA — <GroupName>"`, and `object_ids` set to the IDs of the relevant objects collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
