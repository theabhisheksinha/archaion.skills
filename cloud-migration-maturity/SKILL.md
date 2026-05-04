---
name: cloud-migration-maturity
description: Assess the cloud readiness and migration maturity of an application using CAST Imaging. Identifies cloud blockers, modernization opportunities, and provides a structured migration roadmap.
---

# Cloud Migration Maturity Analysis

Assess the cloud readiness and migration maturity of an application using CAST Imaging.

## 🛠 Tool Resolution (MANDATORY)
This skill uses tools provided by the CAST Imaging MCP server. Because users may rename their MCP server, the tool names in this workflow are referenced by their **base names** (e.g., `advisors`, `quality_insights`).
- **The Agent MUST:** Search the available tools for those containing the keywords "imaging" and the base name.
- **Prefix Handling:** Resolve the correct prefix (e.g., `mcp_imaging_linux_` or `your_server_name_`) by checking active MCP tools before execution.

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

Produce a report structured as follows:

```markdown
## Cloud Migration Maturity Report: [Application Name]

### Executive Summary
[Provide a high-level cloud readiness score (e.g., 0-100% or Low/Medium/High) and a summary of the migration effort.]

### Critical Cloud Blockers
| Rule | Description | Impacted Objects | Severity |
|------|-------------|------------------|----------|
| [Rule Name] | [Why it blocks cloud] | [Object names/counts] | Critical |

### Recommended Migration Strategy
[Choose one: Rehost (Lift & Shift), Replatform (Fix & Finish), or Refactor (Full Modernization).]
```

## 💡 Example Prompts for Users
- "Isolate Cloud Blockers for 'Shopizer' and tell me which components still use the local file system."
- "I want to move to the cloud. Analyze the outward dependencies of 'LegacyApp' for high-latency connections."
- "What is the cloud readiness score for 'TicketMonster'?"

## Tips for the Agent
- Use `object_details(focus="code")` to see the actual code causing a cloud violation.
- Group violations by component using `architectural_graph`.
