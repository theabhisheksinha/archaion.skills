---
name: architectural-governance
description: Enforce architectural best practices and governance. Audits applications for strict layering, circular dependencies, prohibited cross-component calls, and structural compliance using CAST Imaging.
---

# Architectural Governance & Compliance

Enforce architectural best practices using CAST Imaging. This skill acts as an automated architecture review board, ensuring that applications do not violate intended design patterns (e.g., UI calling Database directly).

## 🛠 Tool Resolution (MANDATORY)
This skill uses tools from the CAST Imaging MCP server. Tool names are referenced by **base names**.
- **The Agent MUST:** Resolve the correct prefix (e.g., `mcp_imaging_linux_` or `your_server_name_`) by checking active MCP tools before execution.

## 🛡️ Strict Guardrails & Hallucination Prevention
... (existing guardrails) ...
4.  **No Unverified Claims:** Any reported violation of governance MUST be backed by a specific object ID, file path, or edge returned by the MCP tools.

## 📂 Local Workspace Integration (Path Mapping)
To provide real compliance evidence, the Agent MUST map Imaging paths to the local workspace:
1.  **Establish `local_root`:** Map the Imaging source path to your local filesystem.
2.  **Evidence Collection:** For every governance violation (e.g., prohibited call), use `read_file` to fetch the actual import or call from the local code and include it as evidence in the report.

## Workflow

### 1. Audit Component Layering & Coupling
Ensure components are properly layered and not tightly coupled.
- Call the `architectural_graph` tool (level="component").
- Look for **circular dependencies** (Component A calls Component B, and B calls A).
- Look for **layer skipping** (e.g., Presentation layer components calling Data Access components directly, bypassing Business Logic).

### 2. Verify Prohibited Interactions
Check if specific, unwanted dependencies exist.
- Call the `objects_relationships` tool, providing the IDs of objects that *should not* communicate (e.g., a specific frontend controller and a core database table).
- If the tool returns edges between them, flag this as a Governance Violation.

### 3. Audit Structural Flaws (Maintainability)
Ensure the codebase remains maintainable and adheres to structural standards.
- Call the `quality_insights` tool (nature="structural-flaws").
- Focus on violations related to "High Coupling", "Low Cohesion", or "Dead Code".
- Use the `quality_insight_violations` tool (id="<rule_id>", include_locations=True) to pinpoint exact violations.

### 4. Review External Dependencies
Ensure the application is not exposing or relying on unauthorized external services.
- Call the `inter_applications_dependencies` tool (interaction_types="outward").
- Flag any unknown or legacy applications that violate current IT strategy.

### 5. Generate Governance Audit Report

Produce a report structured as follows:

```markdown
## Architectural Governance Audit: [Application Name]

### 1. Layering & Coupling Violations
| Violation Type | Components Involved | Description | Severity |
|----------------|---------------------|-------------|----------|
| [e.g., Circular Dependency] | [Comp A <-> Comp B] | [Explanation] | High |
| [e.g., Layer Bypass] | [UI -> DB] | [Explanation] | Critical |

### 2. Prohibited Interactions Detected
[List any specific object-to-object calls that violate enterprise standards.]

### 3. Structural Health (Maintainability)
| Governance Rule | Total Violations | Top Offending Object |
|-----------------|------------------|----------------------|
| [Rule Name] | [Count] | [Object Name] |

### 4. External Dependency Compliance
[List approved vs. unapproved external application dependencies.]

### Recommended Remediation
[Provide architectural refactoring advice based STRICTLY on the MCP data.]
```

## 💡 Example Prompts for Users
- "Run an Architectural Governance audit on 'TicketMonster' and find any circular dependencies between components."
- "Verify that the UI layer in 'Shopizer' is not calling the Database layer directly."
- "Check 'LegacyApp' for high-coupling structural flaws that violate our maintainability standards."

## Tips for the Agent
- Use `pathfinder_hierarchy_details` (focus="pathfinder") to prove exactly *how* a prohibited interaction is occurring if a user asks for proof.
