---
name: cloud-migration-maturity
description: Assess the cloud readiness and migration maturity of an application using CAST Imaging. Identifies cloud blockers, modernization opportunities, and provides a structured migration roadmap.
---

# Cloud Migration Maturity Analysis

Assess the cloud readiness and migration maturity of an application using CAST Imaging. This skill identifies technical blockers (e.g., hardcoded IP addresses, local file system usage) and provides recommendations for moving to cloud platforms like AWS, Azure, or GCP.

## Workflow

### 1. Discover Cloud Advisors
Start by identifying which cloud-specific migration advisors are available for the application.
- Call `mcp_imaging_linux_advisors(application="<app_name>", focus="list")`.
- Look for advisors containing names like "Move to AWS", "Move to Azure", "Move to GCP", or "Cloud Readiness".

### 2. Analyze Cloud Blockers
For each relevant cloud advisor found:
- Call `mcp_imaging_linux_advisors(application="<app_name>", focus="rules", advisor_id="<id>")`.
- Identify rules categorized as "Blockers" or "High Priority".
- For critical rules, call `mcp_imaging_linux_advisors(application="<app_name>", focus="violations", advisor_id="<id>", rule_id="<rule_id>")` to find the specific code objects.

### 3. Check Cloud Detection Patterns
- Call `mcp_imaging_linux_quality_insights(application="<app_name>", nature="cloud-detection-patterns")`.
- Use `mcp_imaging_linux_quality_insight_violations(application="<app_name>", nature="cloud-detection-patterns", id="<insight_id>", include_locations=True)` for detailed locations.

### 4. Evaluate External Dependencies
- Call `mcp_imaging_linux_inter_applications_dependencies(application="<app_name>", interaction_types="outward")`.

### 5. Utilize AI Summaries
- Call `mcp_imaging_linux_transaction_details(application="<app_name>", id="<transaction_id>", focus="summary")`.

### 6. Generate Cloud Maturity Report
[Standard report structure follows...]
