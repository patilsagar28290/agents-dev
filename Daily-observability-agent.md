# Daily Observability & Root Cause Analysis (RCA) MCP Agent Setup Guide

This comprehensive technical guide outlines how to build, configure, and automate an autonomous **Model Context Protocol (MCP)** agent workflow. Connected to **GitHub Copilot**, **Dynatrace**, and **Elasticsearch / Kibana**, this agent automatically investigates yesterday's failed user sessions, correlates log stack traces by Correlation ID (CID), performs AI-driven root cause analysis, and delivers a styled HTML report directly to your inbox upon daily system login.

---

## 1. Architecture & System Workflow

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚                      Trigger: Daily System Login                        â”‚
â”‚             (macOS/Linux Shell Profile or Windows Task Scheduler)        â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                     â”‚
                                     â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚                     Daily Agent Python Orchestrator                     â”‚
â”‚                        (Executes Copilot CLI Agent)                     â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                     â”‚
                                     â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚                      GitHub Copilot Agent Engine                        â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                   â”‚                                  â”‚
                   â–¼                                  â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”  â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚      Dynatrace MCP Server        â”‚  â”‚    Elasticsearch MCP Server      â”‚
â”‚  - Fetches failure events (24h)  â”‚  â”‚  - Queries logs by CID & Session â”‚
â”‚  - Extracts Session IDs & CIDs   â”‚  â”‚  - Pulls stack traces & payloads â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜  â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                   â”‚                                  â”‚
                   â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                     â”‚
                                     â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚                      LLM Correlation & RCA Engine                       â”‚
â”‚      - Maps user impact to backend application stack trace exceptions   â”‚
â”‚      - Identifies exact root cause & synthesizes actionable code fix    â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                     â”‚
                                     â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚                        SMTP Email Dispatcher                            â”‚
â”‚                 (Delivers Styled HTML Report to Inbox)                  â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

---

## 2. Prerequisites & Environment Setup

Ensure the following tools and access permissions are configured before setup:

* **Node.js**: `v18.0.0` or higher
* **Python**: `v3.10` or higher
* **GitHub CLI & Copilot Extension**: Installed and logged in (`gh auth login`, `gh extension install github/gh-copilot`)
* **Dynatrace Access Tokens**:
  * Environment API v2 scopes: `events.read`, `userActions.read`, `entities.read`
* **Elasticsearch / Kibana Access**:
  * API Key or Basic Auth with read access to application index patterns (e.g., `logs-*-*` or `app-logs-*`)
* **SMTP Server Access**: Credentials for Gmail App Password, AWS SES, SendGrid, or internal corporate mail server.

---

## 3. Model Context Protocol (MCP) Configuration

Configure the MCP server connections so GitHub Copilot can interact natively with Dynatrace and Elasticsearch APIs.

### Global Configuration Path
* **VS Code / Copilot Global:** `~/.config/Code/User/globalStorage/github.copilot-chat/mcp.json`
* **GitHub Copilot CLI / Claude Desktop:** `~/.config/github-copilot/mcp.json`
* **Project Specific:** `.vscode/mcp.json`

### `mcp.json` Configuration File
Create or update your `mcp.json` file with the following server definitions:

```json
{
  "mcpServers": {
    "dynatrace": {
      "command": "npx",
      "args": [
        "-y",
        "@dynatrace-oss/dynatrace-mcp-server"
      ],
      "env": {
        "DT_TENANT_URL": "https://<your-tenant-id>.live.dynatrace.com",
        "DT_API_TOKEN": "dt0c01.YOUR_DYNATRACE_API_TOKEN_HERE"
      }
    },
    "elasticsearch": {
      "command": "docker",
      "args": [
        "run",
        "-i",
        "--rm",
        "-e", "ES_URL",
        "-e", "ES_API_KEY",
        "docker.elastic.co/mcp/elasticsearch",
        "stdio"
      ],
      "env": {
        "ES_URL": "https://your-elasticsearch-cluster.com:9200",
        "ES_API_KEY": "YOUR_ELASTICSEARCH_API_KEY_HERE"
      }
    }
  }
}
```

---

## 4. Custom GitHub Copilot Prompt (`.github/prompts/daily-rca.prompt.md`)

Store this custom prompt in your repository under `.github/prompts/daily-rca.prompt.md`. This defines the precise multi-step execution path for GitHub Copilot.

```markdown
---
name: Daily RCA Investigation & Email Report
description: Automated daily workflow to fetch yesterday's Dynatrace & Kibana failures, analyze root causes, and draft an HTML report.
tools:
  - dynatrace/*
  - elasticsearch/*
---

# Instructions

You are an automated Site Reliability Engineering (SRE) & Observability Agent. Perform the following analysis:

1. **Step 1: Dynatrace Investigation (Past 24 Hours)**
   - Query Dynatrace for all synthetic and real-user failure events, high error rate metrics, and failed user actions from the last 24 hours.
   - Extract unique **User Session IDs**, **Correlation IDs (CIDs)**, and affected endpoint URLs.

2. **Step 2: Kibana / Elasticsearch Deep Dive**
   - For every extracted CID and Session ID, search Elasticsearch logs for corresponding stack traces, HTTP 5xx responses, and unhandled exceptions.
   - Gather context around payload attributes, service names, and database connection timeouts.

3. **Step 3: Root Cause Synthesis**
   - Map user-facing errors (Dynatrace) to backend application exceptions (Kibana).
   - Determine whether the cause stems from external API timeouts, bad deployments, database bottlenecks, or null pointer exceptions.
   - Formulate a concrete developer-level code fix or configuration change for each incident.

4. **Step 4: HTML Formatting**
   - Output the complete findings directly in a clean, self-contained HTML body template using responsive inline CSS styling.
   - Do NOT wrap output in markdown fences; return raw HTML ready for email sending.
```

---

## 5. Automation Orchestrator (`daily_agent.py`)

Save the following Python script as `daily_agent.py`. This script handles tool execution via GitHub Copilot CLI, formats the output, and dispatches the HTML email report via SMTP.

```python
import os
import sys
import smtplib
import subprocess
from datetime import datetime, timedelta
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

# Configuration via Environment Variables or Defaults
SMTP_SERVER = os.getenv("SMTP_SERVER", "smtp.gmail.com")
SMTP_PORT = int(os.getenv("SMTP_PORT", 587))
SENDER_EMAIL = os.getenv("SENDER_EMAIL", "sre-agent@company.com")
RECEIVER_EMAIL = os.getenv("RECEIVER_EMAIL", "developer@company.com")
SMTP_PASSWORD = os.getenv("SMTP_PASSWORD", "YOUR_SMTP_APP_PASSWORD")

def run_copilot_agent() -> str:
    """Executes the GitHub Copilot CLI prompt to run MCP tools and synthesize RCA."""
    print("ðŸ¤– Launching Copilot MCP Observability Agent...")
    
    prompt_command = (
        "Execute the daily RCA workflow using available Dynatrace and Elasticsearch MCP tools.\n"
        "1. Query yesterday's failed user sessions, error logs, and CIDs from Dynatrace.\n"
        "2. Retrieve associated exception stack traces from Elasticsearch using extracted CIDs.\n"
        "3. Analyze root causes and provide specific code or configuration fixes.\n"
        "4. Format the final output exclusively as a polished, standalone HTML email body with responsive inline styles, summary tables, and dark-themed code snippet blocks for error traces and suggested fixes."
    )
    
    try:
        # Run prompt through Copilot CLI
        result = subprocess.run(
            ["gh", "copilot", "exec", prompt_command],
            capture_output=True,
            text=True,
            check=True
        )
        return result.stdout.strip()
    except subprocess.CalledProcessError as e:
        print(f"âŒ Error executing Copilot CLI: {e.stderr}")
        sys.exit(1)

def send_email_report(html_content: str):
    """Dispatches the HTML report to the target email inbox."""
    yesterday_str = (datetime.now() - timedelta(days=1)).strftime("%Y-%m-%d")
    subject = f"ðŸš¨ Daily Observability RCA Report - {yesterday_str}"

    msg = MIMEMultipart("alternative")
    msg["Subject"] = subject
    msg["From"] = SENDER_EMAIL
    msg["To"] = RECEIVER_EMAIL

    msg.attach(MIMEText(html_content, "html"))

    print(f"ðŸ“§ Connecting to SMTP server ({SMTP_SERVER}:{SMTP_PORT})...")
    try:
        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.starttls()
            server.login(SENDER_EMAIL, SMTP_PASSWORD)
            server.sendmail(SENDER_EMAIL, RECEIVER_EMAIL, msg.as_string())
        print(f"âœ… RCA Report successfully emailed to {RECEIVER_EMAIL}!")
    except Exception as e:
        print(f"âŒ Failed to send email: {e}")

if __name__ == "__main__":
    report_html = run_copilot_agent()
    send_email_report(report_html)
```

---

## 6. Automatic System Login Trigger Setup

Configure your local OS environment to run the agent automatically once per day upon login.

### Option A: macOS / Linux (Zsh or Bash)
Add this logic to your `~/.zshrc` or `~/.bashrc` file. It ensures the script runs in the background on the first terminal launch of each day without slowing down shell startup.

```bash
# --- Daily Observability Agent Login Trigger ---
AGENT_SCRIPT_PATH="$HOME/scripts/daily_agent.py"
LOG_FILE="/tmp/daily_agent.log"
LAST_RUN_FILE="/tmp/daily_agent_last_run"

TODAY=$(date +%Y-%m-%d)
LAST_RUN=$(cat "$LAST_RUN_FILE" 2>/dev/null)

if [ "$TODAY" != "$LAST_RUN" ]; then
    echo "$TODAY" > "$LAST_RUN_FILE"
    echo "[$(date)] Starting Daily MCP RCA Agent..." >> "$LOG_FILE"
    python3 "$AGENT_SCRIPT_PATH" >> "$LOG_FILE" 2>&1 &
fi
```

### Option B: Windows (Task Scheduler)
Create an automated background task triggered by account login:

1. Press `Win + R`, type `taskschd.msc`, and press **Enter**.
2. Click **Create Task** in the right sidebar.
3. **General Tab:**
   * Name: `Daily_MCP_RCA_Agent`
   * Select: **Run only when user is logged on**.
4. **Triggers Tab:**
   * Click **New...** $\rightarrow$ Set **Begin the task:** to **At log on**.
5. **Actions Tab:**
   * Click **New...** $\rightarrow$ Action: **Start a program**.
   * Program/script: `python.exe`
   * Add arguments: `C:\Users\YourUsername\Scripts\daily_agent.py`
6. **Conditions Tab:**
   * Check **Start only if the following network connection is available** (Select Any Connection).
7. Click **OK** to save.

---

## 7. Sample HTML Email Output Structure

The agent outputs structured reports following this HTML visual design pattern:

```html
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; background-color: #f8fafc; color: #1e293b; padding: 20px; }
        .container { max-width: 800px; margin: 0 auto; background: #ffffff; border-radius: 8px; padding: 24px; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1); }
        .header { border-bottom: 2px solid #e2e8f0; padding-bottom: 12px; margin-bottom: 20px; }
        .card { border: 1px solid #e2e8f0; border-left: 4px solid #ef4444; border-radius: 6px; padding: 16px; margin-bottom: 16px; background-color: #fff; }
        .badge { background-color: #fee2e2; color: #991b1b; padding: 2px 8px; border-radius: 4px; font-size: 12px; font-weight: bold; }
        .code-box { background-color: #0f172a; color: #f8fafc; padding: 12px; border-radius: 6px; font-family: monospace; font-size: 13px; overflow-x: auto; }
        .fix-box { background-color: #f0fdf4; border: 1px solid #bbf7d0; color: #166534; padding: 12px; border-radius: 6px; margin-top: 8px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h2>ðŸš¨ Daily Failure Analysis & RCA Report</h2>
            <p>Analysis Period: Yesterday | Source: Dynatrace & Kibana MCP</p>
        </div>
        
        <div class="card">
            <h3><span class="badge">CRITICAL</span> HTTP 500 - Payment Processing Failed</h3>
            <p><strong>Session ID:</strong> <code>dt-sess-8921a</code> | <strong>CID:</strong> <code>cid-pay-998124</code></p>
            <p><strong>Dynatrace Symptom:</strong> User action <code>/api/v1/checkout</code> failed with HTTP 500 during payment authorization step.</p>
            
            <h4>Elasticsearch Stack Trace Log:</h4>
            <div class="code-box">
java.lang.NullPointerException: Cannot invoke "com.payment.Gateway.authorize()" because "this.gateway" is null
    at com.service.PaymentService.processTransaction(PaymentService.java:142)
    at com.controller.CheckoutController.checkout(CheckoutController.java:58)
            </div>
            
            <div class="fix-box">
                <strong>ðŸ’¡ Root Cause & Suggested Fix:</strong><br>
                The payment gateway dependency failed dependency injection due to a missing environment configuration secret key. Verify `PAYMENT_GATEWAY_KEY` is loaded into application properties at startup.
            </div>
        </div>
    </div>
</body>
</html>
```

---

## 8. Troubleshooting & Verification

* **Verify MCP Connectivity:** Test server connections manually using the GitHub Copilot CLI command:
  ```bash
  gh copilot exec "List all available Dynatrace and Elasticsearch tools"
  ```
* **Verify Python Dependencies:** Ensure all standard libraries and Python path configurations are active:
  ```bash
  python3 -c "import smtplib, subprocess; print('Setup environment verified!')"
  ```
* **Check Local Agent Execution Logs:**
  ```bash
  cat /tmp/daily_agent.log
  ```
