---
name: performance-iso5055-review
description: Evaluate application performance and structural quality based on ISO-5055 standards. Identifies efficiency bottlenecks, reliability risks, and maintainability hotspots.
---

# Performance & ISO-5055 Structural Review

Assess performance and structural health using CAST Imaging and ISO-5055 standards.

## 🛠 Tool Resolution (MANDATORY)
This skill uses tools from the CAST Imaging MCP server. Tool names are referenced by **base names**.
- **The Agent MUST:** Resolve the correct prefix (e.g., `mcp_imaging_linux_` or `your_server_name_`) by checking active MCP tools before execution.

## Workflow

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
