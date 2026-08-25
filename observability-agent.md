# End-to-End Observability MCP Server: Setup & Architecture Guide

This document provides a complete guide for setting up a custom **Model Context Protocol (MCP)** server integrated with **GitHub Copilot**. The server aggregates telemetry from **Dynatrace**, **Kibana (Elasticsearch)**, and **AWS ECS/CloudWatch**, providing instant root-cause analysis and automated debugging directly inside your IDE.

---

## 1. High-Level Architecture

```
                                +-----------------------------+
                                |  GitHub Copilot (VS Code)   |
                                +--------------+--------------+
                                               |
                                               | Model Context Protocol (stdio)
                                               v
                                +-----------------------------+
                                | Observability MCP Server    |
                                | (TypeScript / Node.js Engine)|
                                +------+-------+-------+------+
                                       |       |       |
                 +---------------------+       |       +---------------------+
                 |                             |                             |
                 v                             v                             v
   +---------------------------+ +---------------------------+ +---------------------------+
   |   Dynatrace Tool Module   | |    Kibana Tool Module    | |   AWS ECS / CW Module     |
   | - Trace / Log Queries     | | - Elasticsearch Queries   | | - Cluster Health Checks   |
   | - Error Stack Ingestion   | | - Correlation Tracking    | | - Container Exit Codes  |
   +---------------------------+ +---------------------------+ +---------------------------+
```

---

## 2. MCP Server Specifications & Capabilities

The server exposes specialized tools to GitHub Copilot:

| Tool Name | Scope & Purpose | Input Parameters | Output |
| :--- | :--- | :--- | :--- |
| `debug_correlation_id` | Queries Dynatrace and Elasticsearch/Kibana simultaneously for end-to-end request tracing. | `correlationId` (string)<br>`timeWindowMinutes` (number) | Structured JSON containing request flow, microservice HTTP status codes, error logs, and stack traces. |
| `analyze_ecs_failures` | Inspects AWS ECS clusters for stopped tasks, container exit codes, OOM kills, and crash reasons. | `clusterName` (string) | Task ARN status, exit codes (`137` OOM, `1` application crash), stop codes, and runtime events. |
| `fetch_cloudwatch_patterns` | Executes CloudWatch Logs Insights queries over ECS log groups to extract systemic error patterns. | `logGroup` (string)<br>`filterPattern` (string) | Matched error logs, timestamp distribution, and repeating failure signatures. |

---

## 3. Step-by-Step Implementation

### Step 3.1: Initialize the Project

Create a dedicated Node.js project directory and install the required dependencies:

```bash
mkdir mcp-observability-server
cd mcp-observability-server

# Initialize Node.js package
npm init -y

# Install Core MCP SDK and Provider Clients
npm install @modelcontextprotocol/sdk             @aws-sdk/client-ecs             @aws-sdk/client-cloudwatch-logs             @elastic/elasticsearch             axios             dotenv

# Install TypeScript Development Dependencies
npm install --save-dev typescript @types/node tsx
npx tsc --init
```

---

### Step 3.2: Complete Server Implementation (`src/index.ts`)

Create `src/index.ts` with full logic for Dynatrace, Elastic, and AWS handlers:

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema
} from "@modelcontextprotocol/sdk/types.js";
import { ECSClient, DescribeTasksCommand, ListTasksCommand } from "@aws-sdk/client-ecs";
import { CloudWatchLogsClient, StartQueryCommand, GetQueryResultsCommand } from "@aws-sdk/client-cloudwatch-logs";
import { Client as ElasticClient } from "@elastic/elasticsearch";
import axios from "axios";
import dotenv from "dotenv";

dotenv.config();

// Initialize SDK Clients
const awsRegion = process.env.AWS_REGION || "us-east-1";
const ecsClient = new ECSClient({ region: awsRegion });
const cwClient = new CloudWatchLogsClient({ region: awsRegion });

const esClient = new ElasticClient({
  node: process.env.KIBANA_ES_URL || "http://localhost:9200",
  auth: { apiKey: process.env.ES_API_KEY || "" }
});

const server = new Server(
  { name: "observability-mcp-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

/**
 * Define Available Tools
 */
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "debug_correlation_id",
      description: "Searches Dynatrace and Kibana logs/traces for a given Correlation ID to spot failure root cause.",
      inputSchema: {
        type: "object",
        properties: {
          correlationId: { type: "string" },
          timeWindowMinutes: { type: "number", default: 60 }
        },
        required: ["correlationId"]
      }
    },
    {
      name: "analyze_ecs_failures",
      description: "Inspects AWS ECS cluster for stopped tasks, container exit codes, crash reasons, and task failures.",
      inputSchema: {
        type: "object",
        properties: {
          clusterName: { type: "string" }
        },
        required: ["clusterName"]
      }
    }
  ]
}));

/**
 * Handle Tool Execution Requests
 */
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  // -------------------------------------------------------------
  // Tool 1: Debug Correlation ID Across Dynatrace & Kibana
  // -------------------------------------------------------------
  if (name === "debug_correlation_id") {
    const cid = String(args?.correlationId);
    
    // Dynatrace Log/Trace Query
    const dtPromise = axios.get(`${process.env.DYNATRACE_TENANT_URL}/api/v2/logs/search`, {
      headers: { Authorization: `Api-Token ${process.env.DYNATRACE_API_TOKEN}` },
      params: { query: `content="${cid}"`, from: "now-2h" }
    })
    .then(res => res.data.results)
    .catch(err => ({ error: `Dynatrace fetch failed: ${err.message}` }));

    // Kibana / Elasticsearch Query
    const esPromise = esClient.search({
      index: "logstash-*",
      body: {
        query: { match: { "correlation_id": cid } },
        sort: [{ "@timestamp": { order: "desc" } }]
      }
    })
    .then(res => res.hits.hits.map(h => h._source))
    .catch(err => ({ error: `Elasticsearch fetch failed: ${err.message}` }));

    const [dtLogs, kibanaLogs] = await Promise.all([dtPromise, esPromise]);

    return {
      content: [{
        type: "text",
        text: JSON.stringify({ correlationId: cid, dynatrace: dtLogs, kibana: kibanaLogs }, null, 2)
      }]
    };
  }

  // -------------------------------------------------------------
  // Tool 2: Analyze AWS ECS Cluster Failure Patterns
  // -------------------------------------------------------------
  if (name === "analyze_ecs_failures") {
    const cluster = String(args?.clusterName);
    
    try {
      // Step A: Fetch Stopped Tasks
      const listTasks = await ecsClient.send(new ListTasksCommand({ cluster, desiredStatus: "STOPPED" }));
      if (!listTasks.taskArns || listTasks.taskArns.length === 0) {
        return {
          content: [{ type: "text", text: `No stopped tasks found in ECS Cluster: ${cluster}` }]
        };
      }

      // Step B: Describe Stopped Tasks for Diagnostic Data
      const taskDetails = await ecsClient.send(
        new DescribeTasksCommand({ cluster, tasks: listTasks.taskArns.slice(0, 10) })
      );
      
      const failures = taskDetails.tasks?.map(task => ({
        taskArn: task.taskArn,
        stoppedReason: task.stoppedReason,
        stopCode: task.stopCode,
        stoppedAt: task.stoppedAt,
        containers: task.containers?.map(c => ({
          name: c.name,
          exitCode: c.exitCode,
          reason: c.reason
        }))
      }));

      return {
        content: [{
          type: "text",
          text: JSON.stringify({ cluster, failureSummary: failures }, null, 2)
        }]
      };
    } catch (err: any) {
      return {
        content: [{ type: "text", text: `AWS ECS Query Failed: ${err.message}` }]
      };
    }
  }

  throw new Error(`Unknown tool requested: ${name}`);
});

// Run MCP Server over Standard I/O (stdio)
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch(console.error);
```

---

## 4. Environment Setup

Create a `.env` file containing configuration keys for your enterprise environments:

```env
# AWS Configuration
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=your_aws_access_key
AWS_SECRET_ACCESS_KEY=your_aws_secret_key

# Dynatrace Endpoint & Token
DYNATRACE_TENANT_URL=https://your-tenant-id.live.dynatrace.com
DYNATRACE_API_TOKEN=dt0c01.sample.token.value

# Kibana / Elasticsearch Connection
KIBANA_ES_URL=https://your-elasticsearch-endpoint:9200
ES_API_KEY=your_es_api_key_here
```

---

## 5. Integrating with GitHub Copilot in VS Code

Configure VS Code to load your custom MCP server automatically.

### Configuration (`.vscode/mcp.json` or `settings.json`)

Add the server definition under `github.copilot.mcpServers`:

```json
{
  "github.copilot.mcpServers": {
    "observability-agent": {
      "command": "npx",
      "args": [
        "tsx",
        "/absolute/path/to/mcp-observability-server/src/index.ts"
      ],
      "env": {
        "AWS_REGION": "ap-south-1",
        "DYNATRACE_TENANT_URL": "https://your-tenant-id.live.dynatrace.com",
        "DYNATRACE_API_TOKEN": "dt0c01.sample.token.value",
        "KIBANA_ES_URL": "https://your-elasticsearch-endpoint:9200",
        "ES_API_KEY": "your_es_api_key_here"
      }
    }
  }
}
```

---

## 6. Real-World Debugging Workflows

Once connected, ask GitHub Copilot in the chat window:

### Scenario 1: Debugging a Failing Transaction via Correlation ID
> **Prompt:**  
> *"Debug request correlation ID `cid-8821-x-9012`. Query Dynatrace and Kibana to trace where the request failed, show me the HTTP status flow, extract the root cause stack trace, and point me to the source file in our codebase that needs fixing."*

### Scenario 2: Diagnosing Infrastructure & Container Crashes
> **Prompt:**  
> *"Analyze our ECS cluster `prod-payment-gateway-cluster`. Identify any stopped tasks or containers with exit codes, analyze failure reasons, and suggest whether we need infrastructure config updates or application code patches."*
