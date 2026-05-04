---
name: db-migration-advisor
description: Guide database migration projects by analyzing how applications interact with their data layer. Identifies hardcoded SQL, table dependencies, and data access patterns (JPA, MyBatis, etc.).
---

# Database Migration Advisor

Guide database migrations using CAST Imaging.

## 🛠 Tool Resolution (MANDATORY)
This skill uses tools from the CAST Imaging MCP server. Tool names are referenced by **base names**.
- **The Agent MUST:** Resolve the correct prefix (e.g., `mcp_imaging_linux_` or `your_server_name_`) by checking active MCP tools before execution.

## 🛡️ Strict Guardrails & Hallucination Prevention
... (existing guardrails) ...
4.  **No Unverified Edits:** Any suggested code edits or remediations MUST directly address the specific file and line number identified by the `quality_insight_violations` or `object_details(focus="code")` tools. Do not suggest arbitrary code changes without structural proof.

## 📂 Local Workspace Integration (Path Mapping)
To provide real database interaction insights, the Agent MUST map Imaging paths to the local workspace:
1.  **Establish `local_root`:** Map the Imaging source path to your local filesystem.
2.  **Verify SQL usage:** When Imaging flags "Hardcoded SQL", use `grep_search` in the local workspace to find the exact string and `read_file` to show the surrounding DAO context.

## Workflow

### 1. Explore Database Inventory
- Call the `application_database_explorer` tool.

### 2. Analyze Table Dependencies (Data Graphs)
- Call the `data_graphs` tool.
- Use the `data_graph_details` tool (id="<id>", focus="type_graph").
- Use the `data_graph_details` tool (id="<id>", focus="summary").

### 3. Identify Data Access Patterns
- Call the `object_profiles` tool.
- Search for procedures: Call the `objects` tool (filters="type:contains:Procedure").

### 4. Search for Hardcoded SQL
- Use the `object_details` tool (focus="code").

### 5. Find Objects Impacted by Table Changes
- Call the `data_graphs_involving_object` tool (filters="name:equals:<table_name>").

### 6. Generate Database Migration Report
[Standard report structure follows...]

## 💡 Example Prompts for Users
- "We are moving to PostgreSQL. Find all Stored Procedures in the 'Billing' module of 'LegacyDB'."
- "What happens to the business logic if I change the 'customer_id' column in the 'Orders' table?"
- "Analyze the data access patterns of 'Shopizer' and tell me how much hardcoded SQL is present."

## Tips for the Agent
- Use `objects_relationships` to prove direct dependencies between code and tables.
- If Stored Procedures exist, mark them as high-risk for migration.
