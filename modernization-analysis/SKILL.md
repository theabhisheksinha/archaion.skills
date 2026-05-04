---
name: modernization-analysis
description: Analyze application architecture to identify modernization candidates. Helps in decoupling monoliths, identifying service boundaries (Mono2Micro), and finding complex hotspots that hinder agility.
---

# Application Modernization Analysis

This skill uses CAST Imaging to visualize dependencies, identify business-logic clusters, and locate complex code that should be refactored or extracted.

## Workflow

### 1. Map High-Level Architecture
- Call `mcp_imaging_linux_architectural_graph(application="<app_name>", level="component")`.

### 2. Identify Complexity Hotspots
- Call `mcp_imaging_linux_objects(application="<app_name>", filters="searchtype:eq:internal")`.
- Focus on the top 10 most complex objects.
- Use `mcp_imaging_linux_pathfinder_hierarchy_details(source="<id>", focus="hierarchy", hops=2)`.

### 3. Identify Service Boundaries (Transaction Mapping)
- Call `mcp_imaging_linux_transactions(application="<app_name>")`.
- Use `mcp_imaging_linux_transaction_details(id="<id>", focus="summary")`.
- Use `mcp_imaging_linux_transaction_details(id="<id>", focus="type_graph")`.

### 4. Locate Dead Code
- Use `mcp_imaging_linux_object_details(focus="inward")`.

### 5. Generate Modernization Report
[Standard report structure follows...]
