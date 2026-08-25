# Unified Observability & Incident Response MCP Server Guide

This document covers the installation, environment management, authentication retrieval, AWS IAM Role assumption, and multi-environment configuration for the **Unified Observability MCP Server**.

---

## 1. Environment & API Credential Retrieval Guide

Obtain the necessary parameters and access tokens from your target telemetry platforms before configuring the server:

### A. Dynatrace
1. **Tenant URL**: Log into Dynatrace. Your tenant URL follows the pattern: `https://<tenant-id>.live.dynatrace.com` (SaaS) or `https://<managed-domain>/e/<environment-id>` (Managed).
2. **API Token**:
   * Navigate to **Settings** -> **Integration** -> **Access tokens** -> **Generate new token**.
   * Assign permissions: `logs.read` (Read logs), `entities.read` (Read entities), and `traces.read` (Read code-level traces).

### B. Kibana / Elasticsearch
1. **Elasticsearch Endpoint**: Open Kibana -> **Management** -> **Dev Tools**, or copy your cluster URL (e.g., `https://es-cluster.company.internal:9200`).
2. **API Key**:
   * Navigate to **Stack Management** -> **API keys** -> **Create API Key**.
   * Alternatively, generate an API key via Dev Tools:
     ```json
     POST /_security/api_key
     {
       "name": "mcp-observability-key",
       "role_descriptors": {
         "mcp-read-role": {
           "cluster": ["monitor"],
           "index": [{ "names": ["logstash-*", "ecs-*"], "privileges": ["read", "view_index_metadata"] }]
         }
       }
     }
     ```

### C. AWS Credentials & Role Assumption
To connect with AWS environments securely (without hardcoding long-lived admin keys):
* **Local Developer Machine**: Authenticate via AWS CLI/SSO (`aws sso login --profile dev-profile`).
* **Cross-Account Access**: Use AWS Security Token Service (STS) to assume an IAM Role (e.g., `arn:aws:iam::123456789012:role/MCP-Observability-Role`).

### D. Microsoft Teams (Graph API)
1. Navigate to **Azure Portal** (`portal.azure.com`) -> **Entra ID** -> **App registrations** -> **New registration**.
2. Note the **Tenant ID** and **Application (client) ID**.
3. Under **Certificates & secrets**, generate a new **Client Secret**.
4. Under **API permissions**, add the following **Application Permissions** and click **Grant Admin Consent**:
   * `User.Read.All`
   * `Chat.Create`
   * `ChatMessage.Send`

---

## 2. Multi-Environment Configuration (`dev`, `uat`, `prod`)

Instead of hardcoding single endpoints, structure your configuration by environment using a dynamic `.env` configuration file or profile mapping.

### Master `.env` File Structure
```env
# Selected Execution Environment: DEV | UAT | PROD
ACTIVE_ENV=PROD

# Target User for Teams Alerts
TEAMS_ALERT_RECIPIENT=dev-oncall@company.com

# MS Teams Graph API Application (Shared across Envs)
TEAMS_TENANT_ID=00000000-0000-0000-0000-000000000000
TEAMS_CLIENT_ID=11111111-1111-1111-1111-111111111111
TEAMS_CLIENT_SECRET=your_azure_client_secret

# ----------------- DEV ENVIRONMENT -----------------
DEV_AWS_REGION=ap-south-1
DEV_AWS_ROLE_ARN=arn:aws:iam::111111111111:role/MCP-Observability-Role
DEV_DYNATRACE_TENANT_URL=[https://dev-tenant.live.dynatrace.com](https://dev-tenant.live.dynatrace.com)
DEV_DYNATRACE_API_TOKEN=dt0c01.DEV_TOKEN_HERE
DEV_KIBANA_ES_URL=[https://es-dev.company.internal:9200](https://es-dev.company.internal:9200)
DEV_ES_API_KEY=DEV_ES_KEY_HERE

# ----------------- UAT ENVIRONMENT -----------------
UAT_AWS_REGION=ap-south-1
UAT_AWS_ROLE_ARN=arn:aws:iam::222222222222:role/MCP-Observability-Role
UAT_DYNATRACE_TENANT_URL=[https://uat-tenant.live.dynatrace.com](https://uat-tenant.live.dynatrace.com)
UAT_DYNATRACE_API_TOKEN=dt0c01.UAT_TOKEN_HERE
UAT_KIBANA_ES_URL=[https://es-uat.company.internal:9200](https://es-uat.company.internal:9200)
UAT_ES_API_KEY=UAT_ES_KEY_HERE

# ----------------- PROD ENVIRONMENT -----------------
PROD_AWS_REGION=ap-south-1
PROD_AWS_ROLE_ARN=arn:aws:iam::333333333333:role/MCP-Observability-Role
PROD_DYNATRACE_TENANT_URL=[https://prod-tenant.live.dynatrace.com](https://prod-tenant.live.dynatrace.com)
PROD_DYNATRACE_API_TOKEN=dt0c01.PROD_TOKEN_HERE
PROD_KIBANA_ES_URL=[https://es-prod.company.internal:9200](https://es-prod.company.internal:9200)
PROD_ES_API_KEY=PROD_ES_KEY_HERE
