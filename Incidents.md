# Enterprise Solution Architecture: Automated Daily Operational Intelligence Agent

## Executive Summary
This architecture defines an enterprise-grade automation framework using **GitHub Copilot Custom Agents** and the **Model Context Protocol (MCP)**. The system queries cross-organizational incident data from **ServiceNow**, enriches context via **Confluence Knowledge Bases**, and automatically dispatches an executive briefing to **Microsoft Teams** upon developer daily login.

---

## 1. High-Level Architecture Framework

```
+-----------------------------------------------------------------------------------+
|                              TRIGGER LAYER                                        |
|   [ User Login Event / OS Startup Script / Serverless Scheduler ]                 |
+-----------------------------------------------------------------------------------+
                                        |
                                        v
+-----------------------------------------------------------------------------------+
|                        ORCHESTRATION LAYER (GitHub Copilot)                       |
|   Custom Agent: @EnterpriseOpsArchitect (.github/agents/ops-architect.agent.md)   |
+-----------------------------------------------------------------------------------+
                                        |
                            JSON-RPC over Stdio Protocol
                                        v
+-----------------------------------------------------------------------------------+
|                           INTEGRATION LAYER (MCP Server)                          |
|   Server: enterprise-ops-mcp (TypeScript / @modelcontextprotocol/sdk)            |
+-----------------------------------------------------------------------------------+
         |                                |                                |
         | REST API                       | CQL Query API                  | HTTP POST
         v                                v                                v
+------------------+             +-------------------+            +------------------+
|    ServiceNow    |             |    Confluence     |            |  MS Teams Hub    |
| (Incidents Core) |             | (Runbooks/KB)     |            | (Adaptive Cards) |
+------------------+             +-------------------+            +------------------+
```

---

## 2. Model Context Protocol (MCP) Server Implementation

Save as `src/mcp-server.ts`. This server handles standard authentication, protocol parsing, and enterprise REST invocations.

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";
import axios from "axios";

const server = new Server(
  { name: "enterprise-ops-mcp", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "fetch_yesterday_incidents",
      description: "Queries P1/P2 incidents logged across the organization within the prior 24-hour window.",
      inputSchema: {
        type: "object",
        properties: {
          assignment_group: { type: "string", description: "Optional assignment group name filter." }
        }
      }
    },
    {
      name: "fetch_confluence_runbooks",
      description: "Executes a CQL search in Confluence for operational runbooks linked to a given service or incident keyword.",
      inputSchema: {
        type: "object",
        properties: {
          search_term: { type: "string", description: "Target service name or error term." }
        },
        required: ["search_term"]
      }
    },
    {
      name: "send_teams_briefing",
      description: "Posts an Adaptive Card payload to the enterprise Microsoft Teams channel via Webhook.",
      inputSchema: {
        type: "object",
        properties: {
          adaptive_card_json: { type: "object", description: "Valid MS Adaptive Card Schema object." }
        },
        required: ["adaptive_card_json"]
      }
    }
  ]
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    if (name === "fetch_yesterday_incidents") {
      const yesterday = new Date();
      yesterday.setDate(yesterday.getDate() - 1);
      const dateStr = yesterday.toISOString().split("T")[0];

      const query = `sys_created_on>=javascript:gs.dateGenerate('${dateStr}','00:00:00')^sys_created_on<=javascript:gs.dateGenerate('${dateStr}','23:59:59')^priority<=2`;

      const response = await axios.get(`${process.env.SERVICENOW_INSTANCE}/api/now/table/incident`, {
        params: { sysparm_query: query, sysparm_limit: 10 },
        headers: { 
          "Authorization": `Basic ${process.env.SERVICENOW_AUTH_TOKEN}`,
          "Accept": "application/json"
        }
      });

      return { content: [{ type: "text", text: JSON.stringify(response.data.result) }] };
    }

    if (name === "fetch_confluence_runbooks") {
      const cql = `text ~ "${args.search_term}" AND type = "page" ORDER BY lastmodified DESC`;
      const response = await axios.get(`${process.env.CONFLUENCE_URL}/wiki/rest/api/content/search`, {
        params: { cql, limit: 3 },
        headers: { "Authorization": `Bearer ${process.env.CONFLUENCE_PAT}` }
      });

      return { content: [{ type: "text", text: JSON.stringify(response.data.results) }] };
    }

    if (name === "send_teams_briefing") {
      await axios.post(process.env.TEAMS_WEBHOOK_URL!, args.adaptive_card_json);
      return { content: [{ type: "text", text: "Teams notification dispatched successfully." }] };
    }

    throw new Error(`Execution error: Tool '${name}' unmapped.`);
  } catch (err: any) {
    return { isError: true, content: [{ type: "text", text: err.message }] };
  }
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main();
```

---

## 3. GitHub Copilot Custom Agent Specification

Save as `.github/agents/ops-architect.agent.md`.

```markdown
---
name: EnterpriseOpsArchitect
description: Autonomous Operations Architect that compiles daily ServiceNow incident summaries and enriches them with Confluence knowledge base solutions.
tools:
  - fetch_yesterday_incidents
  - fetch_confluence_runbooks
  - send_teams_briefing
---

# Operational Directives

You are the Enterprise Operations AI Architect. Your function is to execute the morning incident reporting pipeline automatically.

### Operational Sequence
1. **Fetch Incidents**: Execute `fetch_yesterday_incidents` to obtain yesterday's high-severity records.
2. **Context Enrichment**:
   - Parse each incident's `short_description` and `cmdb_ci` (Configuration Item).
   - Execute `fetch_confluence_runbooks` for each item to retrieve matching mitigation strategies or post-mortems.
3. **Synthesis & Formatting**:
   - Construct a Microsoft Teams Adaptive Card (`v1.4`) with clear grouping by Incident Priority (P1 vs P2).
   - Link identified Confluence documentation directly to the incident summary card.
4. **Publish Briefing**: Pass the payload to `send_teams_briefing`.
```

---

## 4. Environment & Integration Configuration

### GitHub Copilot Workspace Configuration (`.vscode/mcp.json`)

```json
{
  "mcpServers": {
    "enterprise-ops-mcp": {
      "command": "node",
      "args": ["${workspaceFolder}/dist/mcp-server.js"],
      "env": {
        "SERVICENOW_INSTANCE": "https://your-org.service-now.com",
        "SERVICENOW_AUTH_TOKEN": "Basic <BASE64_CREDENTIALS>",
        "CONFLUENCE_URL": "https://your-org.atlassian.net",
        "CONFLUENCE_PAT": "<PERSONAL_ACCESS_TOKEN>",
        "TEAMS_WEBHOOK_URL": "https://outlook.office.com/webhook/v2/<WEBHOOK_ID>"
      }
    }
  }
}
```

---

## 5. Automated Post-Login Execution Scripts

### Windows (PowerShell Script)
Save as `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\run_daily_briefing.ps1`:

```powershell
# Ensure network stack readiness
Start-Sleep -Seconds 10

# Execute agent workflow headlessly via Copilot CLI
gh copilot run `
  --agent EnterpriseOpsArchitect `
  "Run morning operational sequence: Fetch yesterday's incidents, enrich with Confluence runbooks, and dispatch the Adaptive Card to MS Teams."
```

### macOS / Linux (Shell Script)
Save to `~/.config/autostart/daily_ops.sh` or execute via `~/.zshrc`:

```bash
#!/usr/bin/env zsh

# Delay execution for network connectivity
sleep 10

# Invoke Copilot CLI agent command
gh copilot run   --agent EnterpriseOpsArchitect   "Run morning operational sequence: Fetch yesterday's incidents, enrich with Confluence runbooks, and dispatch the Adaptive Card to MS Teams."
```

---

## 6. Microsoft Teams Adaptive Card Schema (Payload Output)

The agent dynamically constructs and issues this JSON payload to the MS Teams Webhook:

```json
{
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "type": "AdaptiveCard",
  "version": "1.4",
  "body": [
    {
      "type": "TextBlock",
      "size": "Large",
      "weight": "Bolder",
      "text": "âš¡ Enterprise Morning Operations Briefing",
      "color": "Attention"
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Execution Window:", "value": "Yesterday (00:00 - 23:59 UTC)" },
        { "title": "Status:", "value": "Action Required" }
      ]
    },
    {
      "type": "Container",
      "style": "warning",
      "items": [
        {
          "type": "TextBlock",
          "weight": "Bolder",
          "text": "INC0049201 - Authentication Gateway Timeout",
          "wrap": true
        },
        {
          "type": "TextBlock",
          "text": "**Priority:** P1 | **CI:** API-Gateway-Prod",
          "isSubtle": true
        },
        {
          "type": "TextBlock",
          "text": "ðŸ”— [Confluence Runbook: API Gateway Recovery Guide](https://confluence.org.com/pages/viewpage.action?pageId=102938)",
          "wrap": true
        }
      ]
    }
  ]
}
```

---

## 7. Operational Best Practices & Governance

* **Zero-Trust Token Management**: OAuth2 refresh tokens or short-lived Personal Access Tokens (PATs) stored securely in system credential vaults (e.g., Azure Key Vault / Windows Credential Manager) rather than hardcoded environment strings.
* **Rate Limiting & Safety Controls**: The MCP server strictly caps incident queries to 10 records per execution to avoid throttling enterprise REST endpoints.
* **Auditing**: All agent interactions are logged to standard output (`stdio`) for SOC/SIEM integration.
