---
name: modernization-analysis
description: Analyze application architecture to identify modernization candidates. Helps in decoupling monoliths, identifying service boundaries (Mono2Micro), and finding complex hotspots that hinder agility.
---

# Application Modernization Analysis

Analyze application architecture using CAST Imaging.

## 🛠 Tool Resolution (MANDATORY)
This skill uses tools from the CAST Imaging MCP server. Tool names are referenced by **base names**.
- **The Agent MUST:** Resolve the correct prefix (e.g., `mcp_imaging_linux_` or `your_server_name_`) by checking active MCP tools before execution.

## Workflow

### 1. Map High-Level Architecture
- Call the `architectural_graph` tool (level="component").

### 2. Identify Complexity Hotspots
- Call the `objects` tool (filters="searchtype:eq:internal").
- Use the `pathfinder_hierarchy_details` tool (source="<id>", focus="hierarchy", hops=2).

### 3. Identify Service Boundaries (Transaction Mapping)
- Call the `transactions` tool.
- Use the `transaction_details` tool (id="<id>", focus="summary").
- Use the `transaction_details` tool (id="<id>", focus="type_graph").

### 4. Locate Dead Code
- Use the `object_details` tool (focus="inward").

### 5. Generate Modernization Report
[Standard report structure follows...]
