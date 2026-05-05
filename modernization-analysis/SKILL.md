---
name: modernization-analysis
description: Analyze application architecture to identify modernization candidates. Helps in decoupling monoliths, identifying service boundaries (Mono2Micro), and finding complex hotspots that hinder agility.
---

# Application Modernization Analysis

Analyze application architecture using CAST Imaging.

## 🛠 Tool Resolution (MANDATORY)
This skill uses tools from the CAST Imaging MCP server. Tool names are referenced by **base names**.
- **The Agent MUST:** Resolve the correct prefix (e.g., `mcp_imaging_linux_` or `your_server_name_`) by checking active MCP tools before execution.

## 🛡️ Strict Guardrails & Hallucination Prevention
... (existing guardrails) ...
4.  **No Unverified Edits:** Any suggested code edits or remediations MUST directly address the specific file and line number identified by the `quality_insight_violations` or `object_details(focus="code")` tools. Do not suggest arbitrary code changes without structural proof.

## 📂 Local Workspace Integration (Path Mapping)
To provide real code insights, the Agent MUST map Imaging paths to the local workspace:
1.  **Establish `local_root`:** Identify the local project root by looking for build files (pom.xml, package.json, etc.).
2.  **Translate Paths:** Replace `§{main_sources}§/` with your `local_root`.
3.  **Verification:** When identifying a complexity hotspot, use `read_file` to inspect the actual class or method. Include REAL code snippets in your modernization suggestions.

## Workflow

### 1. Map High-Level Architecture
- Call the `architectural_graph` tool (level="component").

### 2. Identify Complexity & Debt Hotspots
- Call the `objects` tool (filters="searchtype:eq:internal").
- Use the `pathfinder_hierarchy_details` tool (source="<id>", focus="hierarchy", hops=2).
- **Proactive Tip:** Components with high technical debt are the best candidates for refactoring or decommissioning. Use the **technical-debt-analyzer** skill to find the specific code-level debt within these hotspots.

### 3. Identify Service Boundaries (Transaction Mapping)
- Call the `transactions` tool.
- Use the `transaction_details` tool (id="<id>", focus="summary").
- Use the `transaction_details` tool (id="<id>", focus="type_graph").

### 4. Locate Dead Code
- Use the `object_details` tool (focus="inward").

### 5. Generate Modernization Report
[Standard report structure follows...]

## 💡 Example Prompts for Users
- "Identify functional boundaries in 'TicketMonster' to suggest service boundaries for a microservices migration."
- "Find the top 5 'God Objects' in 'LegacyMonolith' and show their inward call chains."
- "Show me which components of 'MonolithApp' are the most tightly coupled."

## Tips for the Agent
- Use `data_graph_details` to see how business logic clusters around database tables.
- If asked "How do I split this app?", focus on Transaction Mapping.
