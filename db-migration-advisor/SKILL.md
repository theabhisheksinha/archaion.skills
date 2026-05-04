---
name: db-migration-advisor
description: Guide database migration projects by analyzing how applications interact with their data layer. Identifies hardcoded SQL, table dependencies, and data access patterns (JPA, MyBatis, etc.).
---

# Database Migration Advisor

This skill identifies "gravity" in the database layer and helps developers understand the impact of schema changes on application code.

## Workflow

### 1. Explore Database Inventory
- Call `mcp_imaging_linux_application_database_explorer(application="<app_name>")`.

### 2. Analyze Table Dependencies (Data Graphs)
- Call `mcp_imaging_linux_data_graphs(application="<app_name>")`.
- Use `mcp_imaging_linux_data_graph_details(id="<id>", focus="type_graph")`.
- Use `mcp_imaging_linux_data_graph_details(id="<id>", focus="summary")`.

### 3. Identify Data Access Patterns
- Call `mcp_imaging_linux_object_profiles(application="<app_name>")`.
- Search for procedures: `mcp_imaging_linux_objects(filters="type:contains:Procedure")`.

### 4. Search for Hardcoded SQL
- Use `mcp_imaging_linux_object_details(focus="code")`.

### 5. Find Objects Impacted by Table Changes
- Call `mcp_imaging_linux_data_graphs_involving_object(application="<app_name>", filters="name:equals:<table_name>")`.

### 6. Generate Database Migration Report
[Standard report structure follows...]
