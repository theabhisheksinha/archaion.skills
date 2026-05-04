# CAST Imaging AI Expert Skills (v0.3)

This repository contains a suite of specialized AI "Skills" designed to transform general-purpose AI agents into expert Architectural Architects. By leveraging the **CAST Imaging MCP Server**, these skills provide automated workflows for modernization, cloud migration, security auditing, and performance reviews.

## 🚀 Available Skills

| Skill | Focus Area | Primary Use Case |
| :--- | :--- | :--- |
| **Cloud Migration Maturity** | Cloud Readiness | Identify blockers (hardcoded IPs, local FS) for AWS/Azure/GCP. |
| **Modernization Analysis** | Refactoring | Monolith decoupling, Mono2Micro service boundary mapping. |
| **Database Migration Advisor** | Data Layer | Analyze SQL patterns, table dependencies, and stored procedures. |
| **Security Vulnerability Check** | Structural Security | Audit CVEs, insecure coding patterns, and API exposure. |
| **Performance ISO-5055 Review** | Structural Health | Find bottlenecks using ISO-5055 (Efficiency, Reliability). |

---

## 🛠 Prerequisites: Connecting the CAST MCP Server

All skills require a connection to the CAST Imaging MCP Server.

### **Gemini CLI Configuration**
Add the following to your `C:\Users\ASN\.gemini\settings.json`:

```json
"mcpServers": {
  "cast_imaging_mcp": {
    "httpUrl": "https://presales-in.castsoftware.com/mcp",
    "headers": {
      "X-API-KEY": "YOUR_API_KEY_HERE"
    },
    "timeout": 10000
  }
}
```

---

## 💻 How to Use in Gemini CLI

### **Installation**
1. Open your terminal in the `MBE_AI_Skills_v03` folder.
2. Install the skills locally:
   ```powershell
   gemini skills install .\cloud-migration-maturity.skill --scope workspace
   gemini skills install .\modernization-analysis.skill --scope workspace
   # (Repeat for all .skill files)
   ```
3. Reload the agent:
   ```powershell
   /skills reload
   ```

### **Example Commands**
* *"Assess the cloud readiness of the 'Shopizer' application."*
* *"Find service boundaries in 'TicketMonster' for a microservices migration."*
* *"Perform a security audit of 'WebGoat_v3' focusing on structural flaws."*

---

## 🧑‍💻 How to Use in IDEs

### **VS Code (via Gemini Extension)**
1. **Configure Path:** In VS Code Settings, set `Gemini > Skills: Path` to the folder containing these skills.
2. **MCP Setup:** Add the MCP Server URL and API Key in the Gemini Extension settings panel.
3. **Usage:** Mention the skill in the chat: *"@security-vulnerability-check analyze my current project."*

### **Trae / Cursor / Windsurf**
These IDEs use "Rules" files to guide the agent.
1. **Connect MCP:** Go to **Settings > MCP** and add the HTTP server URL and Header.
2. **Setup Rules:** Create a `.traerules` (or `.cursorrules`) file in your project root.
3. **Inject Workflow:** Copy the `Workflow` section from the desired `SKILL.md` into your rules file.
4. **Usage:** Ask the chat: *"Follow my project rules to perform a database migration analysis."*

---

## 📖 Skill Structure
Each skill folder contains:
*   `SKILL.md`: The expert instruction manual, tool definitions, and reporting templates.
*   `.skill`: The packaged version for distribution and CLI installation.

## ⚠️ Important Note
These skills use the `mcp_imaging_linux_` tool prefix to ensure compatibility with standard CAST Imaging MCP deployments. Ensure your MCP server tools are named accordingly.

---

## 🌍 Open Source & Community Contribution

This project is open-source! We encourage architects and developers to modify these skills, create new ones, and share them back with the community.

### **Sharing via Git**
To share these skills with your team or the public:
1. **Initialize a Repo:** `git init` in this folder.
2. **Commit everything:** Include the `SKILL.md` files and the `.skill` packages.
3. **Push to GitHub/GitLab:** Your team can then `git clone` the repo and have immediate access to the expert workflows.

### **How to Modify and Rebuild Skills**
If you want to customize a workflow (e.g., change the report format or add new tool calls):

1.  **Edit the Source:** Open the `SKILL.md` file in the relevant folder (e.g., `modernization-analysis/SKILL.md`) and make your changes.
2.  **Prerequisites for Building:** Ensure you have `gemini-cli` installed. You will need the `package_skill.cjs` script located in your Gemini CLI installation.
3.  **Run the Packager:** Execute the following command to rebuild the `.skill` file:
    ```powershell
    # Template command
    node path/to/package_skill.cjs path/to/skill-folder
    ```
4.  **Re-install locally:**
    ```powershell
    gemini skills install .\modified-skill.skill --scope workspace
    /skills reload
    ```

### **Contributing Back**
1. **Fork** this repository.
2. **Create a Feature Branch** for your new skill or improvement.
3. **Submit a Pull Request** with a description of the architectural problem your change solves.

---

## 📄 License
This project is shared under the MIT License. Feel free to use, modify, and distribute.
