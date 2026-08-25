# Confluence Deep Search Agent for GitHub Copilot (MCP Integration)

This guide provides a comprehensive setup for integrating a Confluence Model Context Protocol (MCP) server with **GitHub Copilot Chat** (in VS Code, Visual Studio, or JetBrains). This allows GitHub Copilot in **Agent Mode** to deep-search your organization's Confluence spaces, inspect page contents, and filter by metadata using **Confluence Query Language (CQL)**.

---

## Architecture Overview

```
+------------------------+        MCP (Stdio)        +--------------------------+        REST API / CQL        +-------------------+
|  GitHub Copilot Chat   | <-----------------------> | Confluence MCP Server    | <--------------------------> | Confluence Cloud  |
|     (Agent Mode)       |                           | (Node.js or Python)      |                              | (Atlassian REST)  |
+------------------------+                           +--------------------------+                              +-------------------+
```

---

## Prerequisites

1. **Atlassian API Token**:
   * Log in to [Atlassian API Tokens](https://id.atlassian.com/manage-profile/security/api-tokens).
   * Click **Create API Token**, assign a descriptive name (e.g., `GitHub Copilot MCP`), and copy the generated token.
2. **GitHub Copilot**:
   * Ensure GitHub Copilot is enabled in VS Code / Visual Studio / JetBrains and **Agent Mode** is accessible.
3. **Runtime Environment**:
   * **Node.js** (v18+) for Option 1, OR **Python** (3.10+) for Option 2.

---

## Option 1: Quick Setup using Standard Node Package

If you want a pre-built Node.js MCP server, configure your workspace or global MCP settings for Copilot.

### Step 1: Create configuration file

Create or edit the `.mcp.json` file in your repository root (or workspace `.vscode/mcp.json`):

```json
{
  "inputs": [
    {
      "id": "confluence_api_token",
      "type": "promptString",
      "description": "Enter your Atlassian API Token",
      "password": true
    }
  ],
  "servers": {
    "confluence": {
      "command": "npx",
      "args": ["-y", "@aashari/mcp-server-atlassian-confluence"],
      "env": {
        "ATLASSIAN_SITE_NAME": "your-company-domain",
        "ATLASSIAN_USER_EMAIL": "your.email@company.com",
        "ATLASSIAN_API_TOKEN": "${input:confluence_api_token}"
      }
    }
  }
}
```

> **Note**: Replace `your-company-domain` (e.g., if your URL is `https://acme.atlassian.net`, use `acme`) and `your.email@company.com` with your actual Atlassian login email.

---

## Option 2: Custom Python MCP Server with Advanced CQL Deep Search

For full control over Confluence Query Language (CQL) queries, page filtering, and content extraction, build a dedicated Python MCP server using `FastMCP`.

### Step 1: Install Dependencies

```bash
pip install mcp requests
```

### Step 2: Implement `confluence_mcp.py`

Save the following file in your project directory as `confluence_mcp.py`:

```python
import os
import requests
from mcp.server.fastmcp import FastMCP

# Initialize FastMCP Server
mcp = FastMCP("Confluence Deep Search Agent")

# Configuration from Environment Variables
CONFLUENCE_URL = os.getenv("CONFLUENCE_URL", "https://your-company.atlassian.net/wiki").rstrip("/")
EMAIL = os.getenv("CONFLUENCE_EMAIL")
API_TOKEN = os.getenv("CONFLUENCE_API_TOKEN")

AUTH = (EMAIL, API_TOKEN) if EMAIL and API_TOKEN else None
HEADERS = {"Accept": "application/json"}


@mcp.tool()
def search_confluence_cql(cql_query: str, limit: int = 10) -> str:
    """
    Executes a Confluence Query Language (CQL) search for deep/targeted searches.

    Args:
        cql_query: Valid CQL query string.
                   Examples:
                     - 'text ~ "oauth2" AND space = "DEV"'
                     - 'title ~ "Architecture" AND lastmodified >= "2026-01-01"'
                     - 'label = "runbook" AND text ~ "kubernetes"'
        limit: Max results to return (default 10).
    """
    if not AUTH:
        return "Error: Authentication credentials missing. Please set CONFLUENCE_EMAIL and CONFLUENCE_API_TOKEN."

    url = f"{CONFLUENCE_URL}/rest/api/search"
    params = {"cql": cql_query, "limit": limit}

    try:
        response = requests.get(url, auth=AUTH, headers=HEADERS, params=params, timeout=15)
        if response.status_code != 200:
            return f"Error {response.status_code}: {response.text}"

        data = response.json()
        results = data.get("results", [])

        if not results:
            return f"No Confluence pages found matching CQL: `{cql_query}`"

        output = []
        for index, item in enumerate(results, start=1):
            content = item.get("content", {})
            title = content.get("title", "Untitled")
            page_id = content.get("id", "N/A")
            web_url = f"{CONFLUENCE_URL}{item.get('url', '')}"
            excerpt = item.get("excerpt", "No excerpt available.").replace("@@hl@@", "**").replace("@@endhl@@", "**")

            output.append(
                f"### {index}. {title}
"
                f"- **ID**: `{page_id}`
"
                f"- **URL**: {web_url}
"
                f"- **Excerpt**: {excerpt}
"
            )

        return "
".join(output)

    except Exception as err:
        return f"Failed to execute CQL search: {str(err)}"


@mcp.tool()
def get_page_content(page_id: str) -> str:
    """
    Retrieves full content/body of a specific Confluence page by ID.

    Args:
        page_id: The numerical Confluence Page ID.
    """
    if not AUTH:
        return "Error: Authentication credentials missing."

    url = f"{CONFLUENCE_URL}/rest/api/content/{page_id}?expand=body.storage,version"

    try:
        response = requests.get(url, auth=AUTH, headers=HEADERS, timeout=15)
        if response.status_code != 200:
            return f"Error {response.status_code}: {response.text}"

        data = response.json()
        title = data.get("title")
        version = data.get("version", {}).get("number")
        body_html = data.get("body", {}).get("storage", {}).get("value", "")

        return (
            f"# {title} (v{version})
"
            f"**Page ID:** {page_id}

"
            f"## Content (Storage Format):

"
            f"{body_html}"
        )

    except Exception as err:
        return f"Failed to retrieve page content: {str(err)}"


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Step 3: Configure `.mcp.json` for GitHub Copilot

Create or update `.mcp.json` in your repository root or workspace `.vscode/mcp.json`:

```json
{
  "inputs": [
    {
      "id": "confluence_api_token",
      "type": "promptString",
      "description": "Enter Atlassian API Token",
      "password": true
    }
  ],
  "servers": {
    "confluence_cql": {
      "command": "python",
      "args": ["${workspaceFolder}/confluence_mcp.py"],
      "env": {
        "CONFLUENCE_URL": "https://your-company.atlassian.net/wiki",
        "CONFLUENCE_EMAIL": "your.email@company.com",
        "CONFLUENCE_API_TOKEN": "${input:confluence_api_token}"
      }
    }
  }
}
```

---

## CQL (Confluence Query Language) Reference Guide

When asking GitHub Copilot to search, Copilot converts natural language intent into CQL queries:

| Search Intent | Example Natural Prompt to Copilot | Generated CQL |
| :--- | :--- | :--- |
| **Search by Text & Space** | *"Find documentation on OAuth authentication in the DEV space."* | `space = "DEV" AND text ~ "OAuth authentication"` |
| **Search Title** | *"Find any page titled 'Deployment Pipeline'."* | `title ~ "Deployment Pipeline"` |
| **Date Range Filter** | *"Search for incident post-mortems updated after Jan 2026."* | `text ~ "post-mortem" AND lastmodified >= "2026-01-01"` |
| **Label Search** | *"Find all architecture pages tagged with 'runbook'."* | `label = "runbook" AND text ~ "architecture"` |
| **User/Contributor Search** | *"Find pages created by john.doe@company.com."* | `creator = "john.doe@company.com"` |

---

## Usage Instructions in GitHub Copilot

1. Open **GitHub Copilot Chat** in VS Code or your IDE.
2. Select **Agent Mode** from the bottom mode picker dropdown in the chat pane.
3. Verify that the `confluence` or `confluence_cql` tool is listed in available MCP tools.
4. Execute queries like:
   * `@agent Deep search Confluence for production deployment checklists in space 'OPS'.`
   * `@agent Retrieve full content of Confluence page ID 123456789 and summarize the security requirements.`
   * `@agent Find all Confluence pages updated this month with label 'api-spec'.`

---

## Security & Best Practices

1. **Token Protection**: Never hardcode API tokens directly inside `.mcp.json` or commit them to version control. Always use `${input:...}` or local environment variables.
2. **Access Control**: The API token respects the permissions of the Atlassian user account that created it.
3. **Ignore Configuration**: Add sensitive local scripts or environment overrides to `.gitignore`.
