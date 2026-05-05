---
name: technical-debt-analyzer
description: Identifies and prioritizes high-order technical debt in an application. Use this skill when a user wants to reduce technical debt, improve code quality, or identify the top 20 violations across ISO-5055 characteristics (Efficiency, Security, Robustness, Maintainability) with evidence and recommended fixes.
---

# Technical Debt Analyzer

## Overview
This skill provides a systematic approach to identifying the most critical technical debt in an application. It focuses on the "Top 20" violations categorized by ISO-5055 standards, providing evidence-based insights (locations and code snippets) and actionable remediation advice.

## Workflow

### 1. Application Context
First, ensure you have the correct application name. If not provided, ask the user or list applications using `applications()`.

### 2. High-Level Quality Discovery
Use these tools to understand the overall distribution of weaknesses:
- `quality_insights(application, nature="iso-5055")`: Get the count of violations across Efficiency, Security, Robustness, and Maintainability.
- `application_iso_5055_explorer(application)`: List all ISO characteristics to see which ones have the highest violation counts.

### 3. Identify Top Violations
Identify the most critical patterns (Blockers/High Criticality).
- For each high-count characteristic, call `application_iso_5055_explorer(application, characteristic_id="...")` to get the specific weaknesses.
- Pick the weaknesses with the highest impact or count to form the "Top 20" list.

### 4. Gather Evidence for the Top 20
For each of the selected top violations:
- Call `quality_insight_violations(application, nature="iso-5055", id="WEAKNESS_ID", include_locations=True)` to get file names and line numbers.
- For the most representative violations, call `object_details(application, filters="id:eq:OBJECT_ID", focus="code")` to retrieve the actual code snippets.

### 5. Generate the Technical Debt Report
Organize the findings into a report with the following structure:

#### [ISO Characteristic] (e.g., Security)
- **Violation Name**: Brief description of the weakness.
- **Evidence**: File path and line number.
- **Code Snippet**: The offending code (if available).
- **Impact**: Why this matters (e.g., "Exposes system to SQL injection").
- **Recommended Fix**: Specific steps to remediate. Refer to [remediation_patterns.md](references/remediation_patterns.md) for common solutions.

## Guidelines
- **Prioritization**: Always prioritize "Blocker" and "High" criticality violations.
- **Diversity**: Try to include violations from all four ISO-5055 characteristics to give a balanced view of technical debt.
- **Actionability**: Every identified violation MUST have a clear, actionable recommendation.

## Resources
- **[remediation_patterns.md](references/remediation_patterns.md)**: A catalog of common fixes for technical debt patterns.
