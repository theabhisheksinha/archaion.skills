---
name: cloud-migration-maturity
description: Assess the cloud readiness and migration maturity of an application using CAST Imaging. Identifies cloud blockers, modernization opportunities, and provides a structured migration roadmap.
---

# Cloud Migration Maturity Analysis

Assess the cloud readiness and migration maturity of an application using CAST Imaging.

## 🛠 Tool Resolution (MANDATORY)
This skill uses tools provided by the CAST Imaging MCP server. Because users may rename their MCP server, the tool names in this workflow are referenced by their **base names** (e.g., `advisors`, `quality_insights`).
- **The Agent MUST:** Search the available tools for those containing the keywords "imaging" and the base name.
- **Prefix Handling:** If the tools are exposed as `my_server_advisors`, use that full name. If they are `mcp_imaging_linux_advisors`, use that.

## Workflow

### 1. Discover Cloud Advisors
- Identify and call the `advisors` tool (focus="list").
- Look for advisors containing "Move to AWS", "Move to Azure", "Move to GCP", or "Cloud Readiness".

### 2. Analyze Cloud Blockers
- Call the `advisors` tool (focus="rules", advisor_id="<id>").
- Identify "Blockers" or "High Priority" rules.
- Call the `advisors` tool (focus="violations", advisor_id="<id>", rule_id="<rule_id>").

### 3. Check Cloud Detection Patterns
- Call the `quality_insights` tool (nature="cloud-detection-patterns").
- Use the `quality_insight_violations` tool (include_locations=True) for details.

### 4. Evaluate External Dependencies
- Call the `inter_applications_dependencies` tool (interaction_types="outward").

### 5. Utilize AI Summaries
- Call the `transaction_details` tool (id="<id>", focus="summary").

### 6. Generate Cloud Maturity Report
[Standard report structure follows...]
