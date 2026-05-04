---
name: performance-iso5055-review
description: Evaluate application performance and structural quality based on ISO-5055 standards. Identifies efficiency bottlenecks, reliability risks, and maintainability hotspots.
---

# Performance & ISO-5055 Structural Review

This skill identifies the architectural flaws that lead to slow performance and frequent production outages using ISO-5055 characteristics.

## Workflow

### 1. High-Level Quality Assessment
- Call `mcp_imaging_linux_application_iso_5055_explorer(application="<app_name>")`.

### 2. Deep Dive into Efficiency
- Call `mcp_imaging_linux_application_iso_5055_explorer(application="<app_name>", characteristic_id="ISO-5055-Efficiency")`.
- Detailed locations: `mcp_imaging_linux_quality_insight_violations(application="<app_name>", nature="iso-5055", id="<rule_id>", include_locations=True)`.

### 3. Identify Performance Bottlenecks
- Call `mcp_imaging_linux_transaction_profiles(application="<app_name>")`.
- Transaction Summary: `mcp_imaging_linux_transaction_details(id="<id>", focus="summary")`.
- Complexity Graph: `mcp_imaging_linux_transaction_details(id="<id>", focus="complexity_graph")`.

### 4. Maintainability Audit
- Call `mcp_imaging_linux_quality_insights(application="<app_name>", nature="structural-flaws")`.

### 5. Generate ISO-5055 Performance Report
[Standard report structure follows...]
