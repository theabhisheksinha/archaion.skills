---
name: performance-iso5055-review
description: Evaluate application performance and structural quality based on ISO-5055 standards. Identifies efficiency bottlenecks, reliability risks, and maintainability hotspots.
---

# Performance & ISO-5055 Structural Review

Assess performance and structural health using CAST Imaging and ISO-5055 standards.

## 🛠 Tool Resolution (MANDATORY)
This skill uses tools from the CAST Imaging MCP server. Tool names are referenced by **base names**.
- **The Agent MUST:** Resolve the correct prefix (e.g., `mcp_imaging_linux_` or `your_server_name_`) by checking active MCP tools before execution.

## 🛡️ Strict Guardrails & Hallucination Prevention
... (existing guardrails) ...
4.  **No Unverified Edits:** Any suggested code edits or remediations MUST directly address the specific file and line number identified by the `quality_insight_violations` or `object_details(focus="code")` tools. Do not suggest arbitrary code changes without structural proof.

## 📂 Local Workspace Integration (Path Mapping)
To provide real performance profiling, the Agent MUST map Imaging paths to the local workspace:
1.  **Establish `local_root`:** Map the Imaging source path to your local filesystem.
2.  **Inspect Hotspots:** For identified bottlenecks, use `read_file` to show the user the REAL local logic (e.g., the complex loop or unclosed resource).

## Workflow

### 0. Prioritization Tip
- For a consolidated "Top 20" list of the most critical violations across all ISO categories (Efficiency, Security, Robustness, Maintainability), consider using the **technical-debt-analyzer** skill.

### 1. High-Level Quality Assessment
- Call the `application_iso_5055_explorer` tool.

### 2. Deep Dive into Efficiency
- Call the `application_iso_5055_explorer` tool (characteristic_id="ISO-5055-Efficiency").
- Detailed locations: Call the `quality_insight_violations` tool (id="<rule_id>", include_locations=True).

### 3. Identify Performance Bottlenecks
- Call the `transaction_profiles` tool.
- Transaction Summary: Call the `transaction_details` tool (id="<id>", focus="summary").
- Complexity Graph: Call the `transaction_details` tool (id="<id>", focus="complexity_graph").

### 4. Maintainability Audit
- Call the `quality_insights` tool (nature="structural-flaws").

### 5. Generate ISO-5055 Performance Report
[Standard report structure follows...]

## 💡 Example Prompts for Users
- "The 'UserLogin' transaction is slow. Check 'Shopizer' for inefficient loops or database access patterns in that flow."
- "Perform an ISO-5055 structural review of 'TicketMonster' and show me the top reliability risks."
- "Are there any circular dependencies in the 'MainCluster' of tables that could cause deadlocks?"

## Tips for the Agent
- Use `stats` for a quick overview of object counts.
- Focus on the 'Efficiency' characteristic for "Why is my app slow?" questions.
