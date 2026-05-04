---
name: http-client-review
description: Detect and report all outbound HTTP client usage in an application using CAST Imaging. Covers RestTemplate, WebClient, Feign, OkHttp, Apache HttpClient, and other HTTP clients. Includes transaction participation, object counts, violation tracking, and module analysis.
---

# HTTP Client Usage Review

Detect and report all outbound HTTP client usage in an application using the MCP imaging tool. Covers high-level clients, low-level clients, and whether a low-level client is used as the backing transport of a high-level one.

This enhanced version includes:
- URL/access pattern detection (fixed URLs or dynamic URL construction)
- Transaction participation analysis (which transactions use each client)
- Topmost transaction identification (transaction with most objects)
- ISO-5055 violation tracking per transaction
- Module name and package path extraction from fullname

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
      **Filename:** `<app>-http-client-review-<YYYY-MM-DD_HH-mm-ss>.json`
      **Structure:**
      ```json
      {
        "skill": "http-client-review",
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

   This applies to all paginated MCP calls — `mcp__imaging__objects`, `mcp__imaging__transactions_using_object`, `mcp__imaging__transaction_details`, `mcp__imaging__quality_insight_violations`, etc. If a result has `metadata.has_next: true`, fetch all pages, merge items, then write the combined result to the cache file.

3. **Run all detection queries in parallel** — use `mcp__imaging__objects` for each of the following. Read full content (not count-only) to distinguish application code from framework stubs.

   ### Section 1 — RestTemplate
   - `filters="fullname:contains:RestTemplate"`

   ### Section 2 — Spring WebClient
   - `filters="fullname:contains:WebClient"`
   - `filters="fullname:contains:ReactorClientHttpConnector"` ← Reactor Netty backing connector
   - `filters="fullname:contains:JettyClientHttpConnector"` ← Jetty backing connector
   - `filters="fullname:contains:HttpComponentsClientHttpConnector"` ← Apache HttpClient 5 backing connector (reactive)

   ### Section 3 — Feign Client
   - `filters="annotation:contains:FeignClient"`
   - `filters="fullname:contains:FeignClient"`

   ### Section 4 — OkHttp
   - `filters="fullname:contains:OkHttpClient"`
   - `filters="fullname:contains:OkHttp3ClientHttpRequestFactory"` ← OkHttp backing RestTemplate
   - `filters="fullname:contains:feign.okhttp"` ← OkHttp backing Feign

   ### Section 5 — Apache HttpClient
   Distinguish version 4 from version 5 — they use different packages and have different lifecycle status:
   - **HC4** (`org.apache.http`): maintenance mode since 2020, no new features
   - **HC5** (`org.apache.hc.client5`): current actively-developed version

   **HC4 detection:**
   - `filters="fullname:contains:org.apache.http.impl.client"` ← HC4 standalone (`CloseableHttpClient` from HC4 package)
   - `filters="fullname:contains:HttpComponentsClientHttpRequestFactory"` ← HC4 backing RestTemplate
   - `filters="fullname:contains:ApacheHttpClient"` ← HC4 backing Feign (feign-httpclient)

   **HC5 detection:**
   - `filters="fullname:contains:org.apache.hc.client5"` ← HC5 standalone
   - `filters="fullname:contains:HttpComponentsClientHttpConnector"` ← HC5 backing WebClient (reactive)
   - `filters="fullname:contains:ApacheHttp5Client"` ← HC5 backing Feign (feign-hc5)

   **Fallback** (if package path is not available):
   - `filters="fullname:contains:CloseableHttpClient"` — then check the import package in source code to determine HC4 vs HC5

   ### Section 6 — Reactor Netty HttpClient (standalone)
   Reactor Netty can be used directly as an HTTP client, not just as WebClient's backing transport:
   - `filters="fullname:contains:reactor.netty.http.client.HttpClient"` ← standalone Reactor Netty HttpClient
   - If found alongside `WebClient`, check whether it is used independently or only as `ReactorClientHttpConnector` (backing). If the application creates `HttpClient` instances directly (not via `WebClient.builder()`), report as standalone usage.

   ### Section 7 — Retrofit
   Distinguish version 1 (legacy, EOL) from version 2 (current):
   - `filters="fullname:contains:retrofit2."` ← Retrofit 2 (current, actively maintained — package `com.squareup.retrofit2`)
   - `filters="fullname:contains:retrofit."` ← then **exclude** any match also containing `retrofit2.` — remainder is Retrofit 1 (legacy, EOL since 2016, package `retrofit.`)
   - Note: Retrofit always uses OkHttp as its transport. If Retrofit is present, OkHttp is implicitly present even if not directly instantiated in app code.

   ### Section 8 — Other clients
   - `filters="fullname:contains:AsyncHttpClient"` ← AsyncHttpClient (Async-Http-Client / Netty-based)
   - `filters="fullname:contains:HttpURLConnection"` ← Raw JDK HttpURLConnection
   - `filters="fullname:contains:java.net.http.HttpClient"` ← JDK 11+ HttpClient
   - `filters="fullname:contains:UnirestHttp"` ← Unirest

   > **Filtering framework stubs:** Exclude any result whose `filePath` contains `JavaExtractedFiles` or `§<LISA>§` — those are JDK/framework-extracted stubs, not application code. Only objects from the application's own source tree count.

4. **Determine backing client relationships** — for each high-level client found, check whether a corresponding low-level backing client is also present:

   | High-level client | Default backing | Override indicators |
   |-------------------|----------------|---------------------|
   | `RestTemplate` | `SimpleClientHttpRequestFactory` (JDK `HttpURLConnection`) | `HttpComponentsClientHttpRequestFactory` (Apache HC4), `OkHttp3ClientHttpRequestFactory`, `Netty4ClientHttpRequestFactory` |
   | `WebClient` | Reactor Netty (`ReactorClientHttpConnector`) | `JettyClientHttpConnector`, `HttpComponentsClientHttpConnector` (Apache HC5) |
   | `FeignClient` | JDK `HttpURLConnection` (default) | `feign.okhttp.OkHttpClient`, `feign.httpclient.ApacheHttpClient`, `feign.hc5.ApacheHttp5Client` |
   | `Retrofit 2` | OkHttp (always) | — |
   | `Retrofit 1` | OkHttp (always) | — |

   ### Client Maturity Classification

   For each client detected, classify its maturity to help assess whether the technology choice is current or legacy:

   | Client | Era | Status | Assessment |
   |--------|-----|--------|------------|
   | `RestTemplate` | 2009 (Spring 3.0) | Maintenance mode since Spring 5; deprecated in Spring 6 | **Legacy** — migrate to `WebClient` or `RestClient` (Spring 6.1+) |
   | `WebClient` | 2017 (Spring 5.0) | Active, recommended for reactive | **Current** |
   | `RestClient` | 2023 (Spring 6.1) | Active, recommended for blocking | **Current** |
   | `FeignClient` (Spring Cloud OpenFeign) | 2015 | Active | **Current** |
   | `OkHttp` | 2013 | Active (v4 is current) | **Current** |
   | `Apache HttpClient 4` (`org.apache.http`) | 2007 | Maintenance mode since 2020, no new features | **Legacy** — migrate to HC5 |
   | `Apache HttpClient 5` (`org.apache.hc.client5`) | 2020 | Active | **Current** |
   | `Reactor Netty HttpClient` | 2017 | Active, default WebClient transport | **Current** |
   | `Retrofit 2` (`com.squareup.retrofit2`) | 2016 | Active | **Current** |
   | `Retrofit 1` (`retrofit.`) | 2013 | EOL since 2016 | **Obsolete** — migrate to Retrofit 2 |
   | `AsyncHttpClient` | 2010 | Low activity | **Legacy** |
   | `HttpURLConnection` | JDK 1.1 | JDK built-in, no improvements | **Legacy** — use JDK 11+ HttpClient or a framework client |
   | `JDK 11+ HttpClient` (`java.net.http`) | 2018 (JDK 11) | Active, JDK built-in | **Current** |
   | `Unirest` | 2013 | Active | **Current** (lightweight use cases) |

   Include the **Era** and **Status** columns in the report summary table so the reader can immediately see whether a detected client is a modern or legacy choice.

   If a low-level backing client is present alongside a high-level client, report it as a **configured override** rather than independent usage.

5. **Analyze transaction involvement and violations** — for each HTTP client class found in application code, gather transaction and violation data:

   A. **Get transactions using the object:**
   - For each object ID found, call `mcp__imaging__transactions_using_object` with `filters="id:eq:<object_id>"`
   - Extract: `nb_transactions` and `used_in_transactions[]` (contains transaction IDs and startPoint names)

   B. **Get transaction object counts:**
   - For each transaction ID, call `mcp__imaging__transaction_details` with `focus=type_graph`
   - Extract the `size` value from the transaction or sum all node counts to get total objects in the transaction

   C. **Get ISO-5055 violations:**
   - Call `mcp__imaging__quality_insights` with `nature="iso-5055"` to get all violation categories
   - Call `mcp__imaging__quality_insight_violations` with `nature="iso-5055"` to get violation counts per object
   - Map violations to transactions based on objects involved

   D. **Extract module from fullname:**
   - Parse the `fullName` field to extract module name and package path
   - Module is typically the first segment after `com.<organization>` or the Maven artifact name
   - Path is everything after the module name (package structure)

6. **Produce the enhanced report** — use this table structure for all sections:

```
## HTTP Client Usage Report

### Section 1 — RestTemplate
[If absent:] Not used — ✓
[If present:]
| Class | Module | Path (from fullname) | URLs Accessed | # Tx | Top Tx (most objects) | Top Tx Objects | Top Tx Violations | Other High-Violation Tx |
|-------|--------|----------------------|---------------|------|----------------------|----------------|-------------------|------------------------|
| ClassName | module-name | com.package.path | URL1, URL2... | N | tx-name (ID: N) | N | N | tx-name (ID: N) - N violations |

### Section 2 — Spring WebClient
[If absent:] Not used — ✓
[If present:]
| Class | Module | Path (from fullname) | URLs Accessed | # Tx | Top Tx (most objects) | Top Tx Objects | Top Tx Violations | Other High-Violation Tx |
|-------|--------|----------------------|---------------|------|----------------------|----------------|-------------------|------------------------|
| ClassName | module-name | com.package.path | URL1, URL2... | N | tx-name (ID: N) | N | N | tx-name (ID: N) - N violations |

### Section 3 — Feign Client
[If absent:] Not used — ✓
[If present:]
| Interface | Module | Path (from fullname) | URLs Accessed | # Tx | Top Tx (most objects) | Top Tx Objects | Top Tx Violations | Other High-Violation Tx |
|-----------|--------|----------------------|---------------|------|----------------------|----------------|-------------------|------------------------|
| InterfaceName | module-name | com.package.path | URL1, URL2... | N | tx-name (ID: N) | N | N | tx-name (ID: N) - N violations |

### Section 4 — OkHttp
[If absent:] Not used — ✓
[If present — distinguish standalone vs. backing role:]
| Class | Module | Path (from fullname) | Role | URLs Accessed | # Tx | Top Tx (most objects) | Top Tx Objects | Top Tx Violations | Other High-Violation Tx |
|-------|--------|----------------------|------|---------------|------|----------------------|----------------|-------------------|------------------------|
| ClassName | module-name | com.package.path | Standalone/Backing | URL1, URL2... | N | tx-name (ID: N) | N | N | tx-name (ID: N) - N violations |

### Section 5 — Apache HttpClient
[If absent:] Not used — ✓
[If present — distinguish HC4 from HC5 and standalone vs. backing role:]
| Class | Module | Path (from fullname) | Version | Era | Status | Role | URLs Accessed | # Tx | Top Tx (most objects) | Top Tx Objects | Top Tx Violations | Other High-Violation Tx |
|-------|--------|----------------------|---------|-----|--------|------|---------------|------|----------------------|----------------|-------------------|------------------------|
| ClassName | module-name | com.package.path | HC4/HC5 | 2007/2020 | Legacy/Current | Standalone/Backing | URL1, URL2... | N | tx-name (ID: N) | N | N | tx-name (ID: N) - N violations |

### Section 6 — Reactor Netty HttpClient (standalone)
[If absent:] Not used — ✓
[If present — distinguish standalone from WebClient backing:]
| Class | Module | Path (from fullname) | Role | URLs Accessed | # Tx | Top Tx (most objects) | Top Tx Objects | Top Tx Violations | Other High-Violation Tx |
|-------|--------|----------------------|------|---------------|------|----------------------|----------------|-------------------|------------------------|
| ClassName | module-name | com.package.path | Standalone/Backing WebClient | URL1, URL2... | N | tx-name (ID: N) | N | N | tx-name (ID: N) - N violations |

### Section 7 — Retrofit
[If absent:] Not used — ✓
[If present — distinguish v1 (legacy) from v2 (current):]
| Class | Module | Path (from fullname) | Version | Era | Status | URLs Accessed | # Tx | Top Tx (most objects) | Top Tx Objects | Top Tx Violations | Other High-Violation Tx |
|-------|--------|----------------------|---------|-----|--------|---------------|------|----------------------|----------------|-------------------|------------------------|
| ClassName | module-name | com.package.path | v1/v2 | 2013/2016 | Obsolete/Current | URL1, URL2... | N | tx-name (ID: N) | N | N | tx-name (ID: N) - N violations |

### Section 8 — Other Clients
[If none found:] No other HTTP clients detected — ✓
[If present:]
| Client | Class | Module | Path (from fullname) | Era | Status | URLs Accessed | # Tx | Top Tx (most objects) | Top Tx Objects | Top Tx Violations | Other High-Violation Tx |
|--------|-------|--------|----------------------|-----|--------|---------------|------|----------------------|----------------|-------------------|------------------------|
| AsyncHttpClient/JDK HttpClient/HttpURLConnection/Unirest | ClassName | module-name | com.package.path | year | Legacy/Current | URL1, URL2... | N | tx-name (ID: N) | N | N | tx-name (ID: N) - N violations |

### Summary
| HTTP Client | Used | Era | Status | Backing transport | # Tx (Total) | Max Objects in Tx | Total Violations |
|-------------|------|-----|--------|-------------------|--------------|------------------|------------------|
| RestTemplate | Yes / No | 2009 | Legacy | — / Default / Apache HC4 / OkHttp | N | N | N |
| Spring WebClient | Yes / No | 2017 | Current | — / Reactor Netty (default) / Jetty / Apache HC5 | N | N | N |
| Feign Client | Yes / No | 2015 | Current | — / Default / OkHttp / Apache HC4 / Apache HC5 | N | N | N |
| OkHttp (standalone) | Yes / No | 2013 | Current | — | N | N | N |
| Apache HttpClient 4 (standalone) | Yes / No | 2007 | Legacy | — | N | N | N |
| Apache HttpClient 5 (standalone) | Yes / No | 2020 | Current | — | N | N | N |
| Reactor Netty (standalone) | Yes / No | 2017 | Current | — | N | N | N |
| Retrofit 2 | Yes / No | 2016 | Current | OkHttp (always) | N | N | N |
| Retrofit 1 | Yes / No | 2013 | Obsolete | OkHttp (always) | N | N | N |
| Other | Yes / No | — | — | — | N | N | N |
```

Always report all 8 sections. A section with no findings must say "Not used — ✓" rather than be omitted.

### Report Columns Explained

| Column | Description | Source |
|--------|-------------|--------|
| **Module** | Maven/Gradle module name | Extracted from fullname (first segment after org/company) |
| **Path** | Package path from fullname | Everything after module in fullName |
| **URLs Accessed** | Fixed URLs or "dynamic URL construction" if URL is built programmatically | Code analysis + imaging |
| **# Tx** | Number of transactions this client participates in | `nb_transactions` from `transactions_using_object` |
| **Top Tx (most objects)** | Transaction with highest object count | Transaction with max `size` or sum of type_graph nodes |
| **Top Tx Objects** | Number of objects in the top transaction | `size` field or type_graph node counts |
| **Top Tx Violations** | ISO-5055 violations in that transaction | Mapped from quality_insight_violations |
| **Other High-Violation Tx** | Another tx with more violations than top (if exists) | Compare violation counts across transactions |

---

### Problem Summary

> **Usage note:** This section is included when the skill is invoked standalone. When invoked via `/full-review`, this section is omitted — a consolidated summary covering all sections is produced at the end of the global report.

**By discovery order (order checks ran):**

| # | Check | Issues found | Total Tx | Max Tx Objects | Severity |
|---|-------------|-------------------|----------|-----------------|---------|
| 1 | RestTemplate (deprecated in Spring 6) | N class(es) | N | N | ⚠ Legacy / ✓ Not used |
| 2 | Spring WebClient | N class(es) | N | N | ℹ Current / ✓ Not used |
| 3 | Feign Client | N interface(s) | N | N | ℹ Current / ✓ Not used |
| 4 | OkHttp (standalone) | N class(es) | N | N | ℹ Current / ✓ Not used |
| 5a | Apache HttpClient 4 (standalone) | N class(es) | N | N | ⚠ Legacy / ✓ Not used |
| 5b | Apache HttpClient 5 (standalone) | N class(es) | N | N | ℹ Current / ✓ Not used |
| 6 | Reactor Netty HttpClient (standalone) | N class(es) | N | N | ℹ Current / ✓ Not used |
| 7a | Retrofit 2 | N class(es) | N | N | ℹ Current / ✓ Not used |
| 7b | Retrofit 1 | N class(es) | N | N | ⚠ Obsolete / ✓ Not used |
| 8 | Other HTTP clients (HttpURLConnection, JDK HttpClient, AsyncHttpClient…) | N class(es) | N | N | ⚠ Review needed / ✓ Not used |

**By priority (most critical first):**

| Priority | Issue | Detail | Tx Count | Max Objects | Violations | Recommendation |
|----------|---------|--------|----------|-------------|------------|----------------|
| 1 | RestTemplate without configured transport | N class(es) — default JDK HttpURLConnection, no timeout | N | N | N | Configure a `RequestFactory` with timeouts or migrate to `WebClient` / Feign |
| 2 | Raw HttpURLConnection / JDK HttpClient | N class(es) — low-level client with no error handling | N | N | N | Migrate to Spring RestTemplate, WebClient or Feign |
| 3 | RestTemplate (deprecated Spring 6+) | N class(es) | N | N | N | Migrate to `WebClient` (non-blocking) or `RestClient` (blocking, Spring 6.1+) |
| 4 | Multiple coexisting HTTP clients | N different types | N | N | N | Standardise on a single client and a single transport |
| 5 | High transaction involvement | Client in N transactions with M objects | N | M | M | Review for optimization opportunities |

> Replace N values above with actual counts from the analysis. Remove rows where count is 0 and severity is ✓.

---

## Optional — Create a Saved View

After producing the report, if any HTTP clients were detected in application code, offer to create a CAST Imaging saved view for each client type found. Object IDs were already collected during the analysis — no extra queries needed.

Present the offer like this:
```
Would you like me to create a saved view in CAST Imaging with the objects involved?
- RestTemplate usages: N objects
- Feign clients: N objects
Type yes or specify which group(s) to include.
```

For each group the user confirms:
1. Call `mcp__imaging__views` with `focus=create`, `name="HTTP Client — <ClientType>"`, and `object_ids` set to the IDs of the relevant objects collected during the analysis.
2. Return the direct URL from the response so the user can open the view immediately in CAST Imaging.
