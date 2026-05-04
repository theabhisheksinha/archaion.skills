---
name: api-security-review
description: Detect and report API input validation patterns in a Spring application using CAST Imaging. Checks for @Validated on controllers, @Valid on method parameters, bean validation annotations on DTOs, and identifies unvalidated or decoratively-validated endpoints.
---

# API Security Review — Input Validation

Detect and report API input validation patterns in a Spring application using the MCP imaging tool. Checks whether controllers enforce validation (`@Validated`, `@Valid`), whether request body DTOs carry bean validation annotations (`@NotNull`, `@NotBlank`, `@Size`, `@Pattern`, `@Min`, `@Max`, `@Email`, etc.), and identifies endpoints where validation is either missing or decorative (annotation present but no constraints to enforce).

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
      **Filename:** `<app>-api-security-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "api-security-review",
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

3. **Identify all controller classes** — run the following queries in parallel:

   - `mcp__imaging__object_details` with `focus="code"`, `filters="annotation:contains:RestController,type:contains:class"`
   - `mcp__imaging__object_details` with `focus="code"`, `filters="annotation:contains:Controller,type:contains:class"`

   Post-filter: from the `@Controller` results, exclude any class already in the `@RestController` set (since `@RestController` includes `@Controller`). Also exclude classes where the annotation match is a false positive (e.g., `@ComponentScan(basePackages="...controller...")`). Confirm by checking that the `annotations` array contains exactly `"@Controller"`, `"@Controller("`, `"@RestController"`, or `"@RestController("`.

   Record for each controller: `id`, `name`, `fullName`, `annotations`, `filePath`.

4. **Check `@Validated` on controller classes** — for each controller identified in step 3, check if its `annotations` array contains `@Validated` or `@Validated(`.

   Classification:
   | @Validated present? | Meaning |
   |---------------------|---------|
   | Yes | Spring triggers method-level validation for `@RequestParam`, `@PathVariable` constraints, and group-based validation on `@RequestBody`. |
   | **No** | Method parameter constraints (`@Min`, `@Max`, `@Size` on `@RequestParam` / `@PathVariable`) are **silently ignored** — they have no effect without class-level `@Validated`. `@Valid` on `@RequestBody` still works (it triggers bean validation), but `@Validated` with groups does not. |

   > **Key distinction:** `@Valid` on a `@RequestBody` parameter triggers bean validation regardless of class-level `@Validated`. But `@Validated` on the class is **required** for method parameter constraints (`@RequestParam @Min(1) int page`) to be enforced. Without it, the `@Min(1)` is purely decorative.

5. **Identify all handler methods** — for each controller class, call `mcp__imaging__source_file_details` with `nature="inventory"` to list all methods. Alternatively, use `mcp__imaging__object_details` with `focus="code"` and `filters="annotation:contains:Mapping,type:contains:method"` (catches `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`, `@RequestMapping`).

   Post-filter for methods whose `annotations` contain any of:
   - `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`, `@RequestMapping`

   Record for each method: `name`, `annotations`, parent class, `filePath`.

6. **Analyze `@Valid` / `@Validated` on method parameters** — for each handler method identified in step 5, read the local source file to inspect parameter annotations. Use `Grep` with `-A 10` context on the method signature (method name from step 5), because method signatures can span multiple lines.

   For each parameter, extract:
   - Parameter annotations: `@RequestBody`, `@RequestParam`, `@PathVariable`, `@ModelAttribute`
   - Validation trigger: `@Valid` or `@Validated` or `@Validated(Group.class)`
   - Constraint annotations directly on the parameter: `@NotNull`, `@NotBlank`, `@Size`, `@Min`, `@Max`, `@Pattern`, `@Positive`, `@Email`, etc.
   - Parameter type: the Java class (e.g., `CreateProductDto`, `String`, `Long`)

   Classify each parameter:

   | Parameter Type | @Valid/@Validated present? | Assessment |
   |---------------|--------------------------|-----------|
   | `@RequestBody SomeDto` | Yes (`@Valid`) | Check step 7 — does `SomeDto` have bean validation annotations? |
   | `@RequestBody SomeDto` | **No** | **Unvalidated request body** — no bean validation triggered |
   | `@RequestParam` / `@PathVariable` with `@Min`, `@Size`, etc. | N/A (class-level `@Validated` needed) | Check step 4 — is class-level `@Validated` present? If not, constraints are **decorative**. |
   | `@RequestParam` / `@PathVariable` without constraints | N/A | No validation — acceptable only if the type itself is safe (e.g., `Long`, `UUID`) |

7. **Inspect request body DTOs for bean validation annotations** — for each DTO class used as a `@RequestBody` parameter with `@Valid`:

   a. Locate the DTO class source file. Use `mcp__imaging__objects` with `filters="name:contains:<ClassName>,type:contains:class"` to find it, or derive the path from the package.

   b. Read the source file and check every field for bean validation annotations:

   **Standard Bean Validation (javax.validation / jakarta.validation):**
   - `@NotNull`, `@NotBlank`, `@NotEmpty`
   - `@Size(min=, max=)`, `@Length`
   - `@Min`, `@Max`, `@Positive`, `@PositiveOrZero`, `@Negative`, `@NegativeOrZero`
   - `@Pattern(regexp=)`
   - `@Email`
   - `@Digits(integer=, fraction=)`
   - `@Past`, `@PastOrPresent`, `@Future`, `@FutureOrPresent`
   - `@DecimalMin`, `@DecimalMax`
   - `@AssertTrue`, `@AssertFalse`

   **Hibernate Validator extensions:**
   - `@URL`, `@Range`, `@CreditCardNumber`, `@ISBN`, `@UUID`

   c. Classify the DTO:

   | Fields with constraints | @Valid present on @RequestBody | Assessment |
   |------------------------|------------------------------|-----------|
   | Some fields annotated | Yes | Validated — constraints will be enforced |
   | **No fields annotated** | Yes | **Decorative @Valid** — `@Valid` triggers validation but the DTO has no constraints to enforce. The validation passes for any input. This is a false sense of security. |
   | Some fields annotated | **No** | **Unvalidated** — DTO has constraints but they are never triggered because `@Valid` is missing on the parameter |
   | No fields annotated | No | No validation — flag if the DTO has mutable String/collection fields that should be constrained |

   d. **Check nested DTOs:** If a DTO field is itself a complex type (another DTO), check if it has `@Valid` on the field declaration. Without `@Valid` on nested objects, validation does not cascade — constraints on the nested DTO's fields are ignored.

   > **Imaging limitation:** Bean validation annotations on DTO fields may not be indexed by Imaging if the DTO is not an `@Entity` or Spring-managed class. Always fall back to reading the local source file for DTO analysis.

8. **Check for common security anti-patterns:**

   **8a. `@RequestBody` without `@Valid` — unvalidated input:**
   - List all handler methods that accept a `@RequestBody` parameter without `@Valid` or `@Validated`.
   - **Why:** The DTO may have bean validation annotations, but they are never triggered. This is the most common validation gap — the developer adds constraints to the DTO but forgets to trigger them.

   **8b. Decorative `@Valid` — empty DTO:**
   - List all `@RequestBody @Valid SomeDto` parameters where `SomeDto` has zero bean validation annotations on its fields.
   - **Why:** `@Valid` gives a false sense of security. The validation framework runs but finds nothing to validate — every input passes.

   **8c. Decorative method parameter constraints — missing class-level `@Validated`:**
   - List all controllers that do NOT have `@Validated` on the class but have methods with constraint annotations on `@RequestParam` / `@PathVariable` parameters (e.g., `@Min`, `@Max`, `@Size`, `@Pattern`, `@NotBlank`).
   - **Why:** Without class-level `@Validated`, Spring does not create a validation proxy for the controller. Method parameter constraints are ignored at runtime — they compile and look correct but have zero effect. This is a silent, dangerous gap.

   **8d. Missing `@Valid` on nested DTO fields:**
   - For each validated DTO (`@RequestBody @Valid SomeDto`), check if any field is a complex type (another class, not a primitive/wrapper/String/collection-of-primitives) and is missing `@Valid`.
   - **Why:** Bean validation does not cascade into nested objects by default. Without `@Valid` on the nested field, constraints on the nested DTO's fields are silently ignored.

   **8e. Partial validation coverage (unconstrained fields on validated DTOs):**
   - For each validated DTO (`@RequestBody @Valid`), compare the total number of mutable fields (exclude `static`, `final`, and framework-injected fields) against the number of fields carrying at least one bean validation annotation.
   - If coverage is < 100%, list the unconstrained fields by name and type.
   - **Why:** Partial coverage creates blind spots. If 3 out of 8 fields are validated, the remaining 5 accept any input — including null, empty, oversized, or malformed values. Attackers target the weakest field, not the strongest. Partial validation is often worse than no validation because it gives a false impression of safety during code review.

   **8f. String parameters without `@Size` or `@Pattern`:**
   - For validated DTOs, list String fields that have `@NotNull` or `@NotBlank` but no `@Size(max=)` or `@Pattern`.
   - **Why:** Without size limits, an attacker can send arbitrarily large strings, causing memory pressure or database truncation errors. `@NotBlank` prevents empty strings but not 10MB strings.

   **8g. Collection/List parameters without `@Size(max=)`:**
   - For validated DTOs, list `List<>` / `Set<>` / `Collection<>` fields without `@Size(max=)`.
   - **Why:** Unbounded collections allow an attacker to send thousands of elements, causing memory exhaustion or slow processing.

   **8h. Enum-like String fields without `@Pattern`:**
   - If a String field represents a fixed set of values (e.g., status, type, role), it should use `@Pattern` or be typed as an `enum`. Flag String fields named `status`, `type`, `role`, `category`, `kind`, `state` that have no `@Pattern` constraint.

9. **Check for global validation error handling** — use `Grep` on `local_root` to check for:

   a. `@ControllerAdvice` or `@RestControllerAdvice` classes that handle `MethodArgumentNotValidException` (thrown when `@Valid` fails) and `ConstraintViolationException` (thrown when `@Validated` method parameter constraints fail).

   b. If no handler exists, validation errors result in a default Spring 400 response that may leak internal field names and constraint messages to the client — flag as an information disclosure risk.

## Report Template

### Overview
| Metric | Count |
|--------|-------|
| Total controller classes | N |
| Controllers with `@Validated` | N |
| Controllers **without** `@Validated` | N |
| Total handler methods (endpoints) | N |
| Endpoints with `@RequestBody` | N |
| `@RequestBody` with `@Valid` | N |
| `@RequestBody` **without** `@Valid` | N |
| Unique request body DTO classes | N |
| DTOs with bean validation annotations | N |
| DTOs **without** bean validation annotations (empty) | N |
| `@ControllerAdvice` validation error handler | Present / **Missing** |

### Controller Validation Status
| Controller | @RestController | @Validated | Endpoints | @RequestBody count | @Valid count | Unvalidated bodies | Decorative param constraints | File |
|-----------|----------------|-----------|-----------|-------------------|-------------|-------------------|----------------------------|------|
| ClassName | Yes/No | Yes/**No** | N | N | N | N | N | path/to/File.java |

### Endpoint-Level Detail
| Controller | Method | HTTP | @RequestBody param | @Valid? | DTO class | DTO has constraints? | Assessment |
|-----------|--------|------|-------------------|--------|-----------|---------------------|-----------|
| ClassName | createProduct | POST | CreateProductDto | Yes | CreateProductDto | Yes (5 fields) | Validated |
| ClassName | updateProduct | PUT | UpdateProductDto | **No** | UpdateProductDto | Yes (3 fields) | **Unvalidated** |
| ClassName | listProducts | GET | — | — | — | — | No body |

### DTO Validation Coverage
| DTO Class | Used by | Total mutable fields | Constrained | Unconstrained | Coverage | @Valid on nested? | Assessment | File |
|----------|---------|---------------------|------------|--------------|----------|------------------|-----------|------|
| CreateProductDto | ProductController.createProduct | 8 | 5 | 3 | 63% | Yes/No/N/A | Partial | path/to/Dto.java |
| UpdateProductDto | ProductController.updateProduct | 6 | 0 | 6 | **0%** | N/A | **Decorative @Valid** | path/to/Dto.java |
| SimpleDto | SimpleController.create | 4 | 4 | 0 | 100% | N/A | Full | path/to/Dto.java |

### Anti-Pattern Detail

#### 8a. @RequestBody without @Valid (unvalidated input)
| Controller | Method | DTO Class | DTO has constraints? | Risk |
|-----------|--------|-----------|---------------------|------|
| ClassName | methodName | DtoClass | Yes → constraints exist but never triggered | **High** |
| ClassName | methodName | DtoClass | No → no constraints to trigger | Medium |

> **Why this matters:** The request body is deserialized but never validated. If the DTO has bean validation annotations, they exist only as documentation — an attacker can send any payload. If the DTO has no annotations, there are no guardrails at all.

#### 8b. Decorative @Valid (empty DTO)
| Controller | Method | DTO Class | DTO fields | Constrained fields | File |
|-----------|--------|-----------|-----------|-------------------|------|
| ClassName | methodName | DtoClass | N | 0 | path/to/Dto.java |

> **Why this matters:** `@Valid` is present, suggesting the developer intended validation. But the DTO has no `@NotNull`, `@Size`, `@Pattern`, or any other constraint annotations. Every payload passes validation — this is a **false sense of security**.

#### 8c. Decorative parameter constraints (missing @Validated on class)
| Controller (no @Validated) | Method | Parameter | Constraint | Effect |
|---------------------------|--------|-----------|-----------|--------|
| ClassName | methodName | @RequestParam @Min(1) page | @Min(1) | **Silently ignored** |

> **Why this matters:** Without `@Validated` on the controller class, Spring does not create a method validation proxy. Constraint annotations on `@RequestParam` and `@PathVariable` parameters compile and look correct but are **never enforced at runtime**. Any value passes through, including negative numbers, empty strings, or oversized inputs that the developer explicitly tried to prevent.

#### 8d. Missing @Valid on nested DTOs
| DTO Class | Field | Field Type (nested DTO) | @Valid? | File |
|----------|-------|------------------------|--------|------|
| ParentDto | address | AddressDto | **No** | path/to/ParentDto.java |

> **Why this matters:** Bean validation does not cascade by default. Without `@Valid` on a nested DTO field, all constraints on `AddressDto`'s fields (`@NotNull street`, `@Size zip`, etc.) are silently ignored, even when the parent DTO is validated.

#### 8e. Partial validation coverage (unconstrained fields on validated DTOs)
| DTO Class | Total mutable fields | Constrained fields | Unconstrained fields | Coverage | File |
|----------|---------------------|-------------------|---------------------|----------|------|
| CreateProductDto | 8 | 3 | 5 (description, category, tags, metadata, notes) | 38% | path/to/Dto.java |

> **Why this matters:** Partial coverage creates blind spots. An attacker targets the unconstrained fields — those that accept any input without validation. During code review, the presence of *some* annotations gives a false impression that the DTO is validated. Every mutable field on a request body DTO should carry at least a `@NotNull` / `@NotBlank` / `@Size` constraint.

#### 8f. String fields without size limits
| DTO Class | Field | Has @NotNull/@NotBlank | Has @Size(max) or @Pattern | File |
|----------|-------|----------------------|--------------------------|------|
| DtoClass | description | Yes | **No** | path/to/Dto.java |

#### 8g. Unbounded collections
| DTO Class | Field | Collection type | Has @Size(max) | File |
|----------|-------|----------------|---------------|------|
| DtoClass | items | List<ItemDto> | **No** | path/to/Dto.java |

#### 8h. Enum-like String fields without @Pattern
| DTO Class | Field name | Likely enum? | Has @Pattern | File |
|----------|-----------|-------------|-------------|------|
| DtoClass | status | Yes (name suggests fixed values) | **No** | path/to/Dto.java |

### Validation Error Handling
| Handler Class | Exception Handled | Response Format | File |
|--------------|-------------------|----------------|------|
| GlobalExceptionHandler | MethodArgumentNotValidException | Custom JSON | path/to/Handler.java |
| — | ConstraintViolationException | **Not handled** — default Spring 400 | — |

> If `ConstraintViolationException` is not handled, method-parameter validation failures (from `@Validated` + `@Min`/`@Size` on `@RequestParam`) return a default 500 error with internal stack traces. A `@ControllerAdvice` handler should catch it and return a clean 400.

## Problem Summary

| # | Check | Issues found | Severity |
|---|-------|-------------|----------|
| 1 | @RequestBody without @Valid | N endpoint(s) | High — unvalidated input |
| 2 | Decorative @Valid (empty DTO) | N endpoint(s) | High — false sense of security |
| 3 | Missing @Validated on controller (decorative param constraints) | N controller(s) with N decorative constraints | High — silently ignored constraints |
| 4 | Missing @Valid on nested DTO fields | N field(s) | Medium — partial validation |
| 5 | Partial validation coverage (unconstrained fields) | N DTO(s) with N unconstrained fields | Medium — blind spots |
| 6 | String fields without @Size(max) | N field(s) | Medium — unbounded input |
| 7 | Unbounded collections | N field(s) | Medium — memory exhaustion risk |
| 8 | Enum-like String without @Pattern | N field(s) | Low — weak typing |
| 9 | Missing validation error handler | MethodArgumentNotValidException / ConstraintViolationException | Medium — information disclosure |

### Priority Fix Order
| Priority | Finding | Recommendation |
|----------|---------|---------------|
| P1 | @RequestBody without @Valid | Add `@Valid` to all `@RequestBody` parameters |
| P1 | Missing @Validated on controllers with param constraints | Add `@Validated` to the controller class |
| P2 | Decorative @Valid (empty DTO) | Add bean validation annotations to DTO fields |
| P2 | Missing @Valid on nested DTOs | Add `@Valid` to nested DTO field declarations |
| P3 | String fields without @Size | Add `@Size(max=N)` to all String fields |
| P3 | Unbounded collections | Add `@Size(max=N)` to all List/Set fields |
| P4 | Missing validation error handler | Add `@ControllerAdvice` handling both exceptions |
