---
name: db-migration-advisor
description: Guide database migration projects by analyzing how applications interact with their data layer. Identifies hardcoded SQL, table dependencies, and data access patterns (JPA, MyBatis, etc.).
---

# Database Migration Advisor

Guide database migrations using CAST Imaging.

## 🛠 Tool Resolution (MANDATORY)
This skill uses tools from the CAST Imaging MCP server. Tool names are referenced by **base names**.
- **The Agent MUST:** Resolve the correct prefix (e.g., `mcp_imaging_linux_` or `your_server_name_`) by checking active MCP tools before execution.

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
