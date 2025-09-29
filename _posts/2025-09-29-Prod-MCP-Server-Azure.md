---
layout: post
title: "Production MCP Server on Azure: Auth, Deploy, Update"
date: 2025-09-29 12:00:00 +0100
categories: Research, Product Development
tags: mcp, azure, container-apps, entra-id, langchain, langgraph
---

#### In this post I’ll show how to secure and ship a Model Context Protocol (MCP) server on Azure Container Apps (ACA) using Azure Entra ID (OAuth2 client-credentials)—and how to iterate safely (add tools, redeploy, test, and promote) without breaking production.

 Repo (server + clients + infra): https://github.com/PrynAI/PrynAI-MCP

### What you’ll build

- A self-hosted MCP server behind Azure Entra ID (Bearer JWT required)

- Deployed on Azure Container Apps with revisions for safe rollouts

- Images built by ACR remote build (no local Docker needed)

- A repeatable update flow (modify server.py → build → new revision → smoke test → promote)

### Prereqs

- Azure subscription + Owner/Contributor to a resource group

- Azure CLI (az) & PowerShell (or Bash)

- Entra ID admin (to create app registrations and grant admin consent)

- Redis (optional but recommended; I use Azure Cache for Redis)

- Clone the repo:
```
git clone https://github.com/PrynAI/PrynAI-MCP.git
cd PrynAI-MCP
```


### 1) Secure the MCP with Entra ID (client-credentials)

- We use two app registrations:

- Server API (exposed resource)

- Register app: e.g., PrynAI-MCP-Server

- Expose an API → set Application ID URI, e.g. api://<server-app-guid>

- Add an App Role (Application type), e.g.:

    - Name: MCP.Access
    - Value: MCP.Access
    - Allowed member type: Applications

- Save the Application (server client) ID and the App ID URI

### Client (the caller/agent)

- Register app: e.g., PrynAI-MCP-Client

- Create a client secret

- API permissions → Add a permission → My APIs → select your Server API

- Under Application permissions choose the app role MCP.Access

- Grant admin consent

You will need these values (used by server and clients):

- ENTRA_TENANT_ID — Directory (tenant) ID

- SERVER_APP_ID_URI — e.g., api://<server-app-guid> (From PrynAI-MCP-Server)

- ENTRA_CLIENT_ID / ENTRA_CLIENT_SECRET — of the client app (From PrynAI-MCP-Client)

On the client, getting a token looks like:
```
import msal, os

TENANT_ID = os.environ["ENTRA_TENANT_ID"]
CLIENT_ID = os.environ["ENTRA_CLIENT_ID"]
CLIENT_SECRET = os.environ["ENTRA_CLIENT_SECRET"]
SERVER_APP_ID_URI = os.environ["SERVER_APP_ID_URI"]  # api://...

app = msal.ConfidentialClientApplication(
    CLIENT_ID,
    authority=f"https://login.microsoftonline.com/{TENANT_ID}",
    client_credential=CLIENT_SECRET,
)
token = app.acquire_token_for_client([f"{SERVER_APP_ID_URI}/.default"])["access_token"]
# Then call MCP with: Authorization: Bearer <token>
```

### 2) Provision Azure (once)

My deployed names (replace if you differ):

- Resource Group: prynai-mcp-rg

- Container Apps Environment: prynai-aca-env

- Container App: prynai-mcp

- Azure Container Registry: prynaiacr44058 (prynaiacr44058.azurecr.io)

- Azure Cache for Redis: prynai-redis (prynai-redis.redis.cache.windows.net)

You can create these with CLI or the portal. The repo includes infra scripts; for first setup you likely used your own flow. The steps below focus on updates.

### 3) First deploy (or update) via ACR remote build

This project ships a remote build + rolling update script:
```
infra/azure/deploy_update.ps1
```

#### What it does:

- Builds the container in ACR (no local Docker)

- Creates a new ACA revision using the new image tag

- Prints a preview FQDN for smoke testing

- Provides a one-liner to promote that revision (shift traffic to 100%)
  
Run
```
- # from repo root
.\infra\azure\deploy_update.ps1
# or pin a tag (e.g. git SHA or version)
.\infra\azure\deploy_update.ps1 -Tag 0.5.0
```

You’ll see output like:
```
ACR remote build: prynaiacr44058  Image: prynai-mcp:<tag>
New revision: prynai-mcp--00000xyz
Preview FQDN: https://prynai-mcp--00000xyz.<env>.eastus.azurecontainerapps.io
Smoke-test the revision:
$env:PRYNAI_MCP_URL = "https://prynai-mcp--00000xyz.../mcp"; 

Promote when ready:
az containerapp ingress traffic set -g prynai-mcp-rg -n prynai-mcp --revision-weight prynai-mcp--00000xyz=100
```

✅ Best practice: ACA multiple revisions let you test the new pod at its own FQDN. Only switch traffic once smoke tests pass.

### 4) Add/modify tools in server.py

- The MCP server exposes tools via MCP protocol.

- Sync or async? Both are fine. The MCP server layer runs them inside an async runtime. Use sync when the function is CPU-light and blocking calls are minimal; use async for I/O (HTTP, Redis, DB).

- ### Example:

- #### server.py (snippets)
```
# Simple sync tool
def add(a: int, b: int) -> str:
    """Return a+b."""
    return str(a + b)

# Async tool (e.g., calls Redis or another HTTP service)
async def echo(text: str) -> str:
    """Echo back the text."""
    return text

# Register in your MCP server factory / startup
server.add_tool("add", add, schema={"a": int, "b": int})
server.add_tool("echo", echo, schema={"text": str})
```

- Ensure each tool has a clear schema (names/types) and concise docstring. Downstream agents infer how to call it from these.

### 5) Redeploy safely (new tools)

- Commit your server.py changes

- Run the remote build again:
```
- .\infra\azure\deploy_update.ps1 
```

- Smoke test the new revision (script prints the FQDN):
```
- $env:PRYNAI_MCP_URL = "https://<new-revision-fqdn>/mcp"
uv run python .\examples\use_core_list_tools.py
```

- Promote when green:
  ```
  az containerapp ingress traffic set `
  -g prynai-mcp-rg `
  -n prynai-mcp `
  --revision-weight <new-revision-name>=100

  ```

- The primary FQDN remains stable, e.g.:
https://prynai-mcp.purplegrass-10f29d71.eastus.azurecontainerapps.io/mcp

### 6) Test from agents (optional)

### The repo includes:

- LangGraph smoke: examples/phase5_langgraph_smoke.py

- LangChain create_agent smoke: examples/phase5_langchain_create_agent_alltools_smoke.py

- Core adapters to turn MCP tools into LangChain tools: src/prynai/mcp_core.py

- All authenticate with Entra ID and use Authorization: Bearer <token>.

### 7) Troubleshooting

- 401: Ensure the client token is issued for SERVER_APP_ID_URI and you used scope = f"{SERVER_APP_ID_URI}/.default". Admin consent must be granted to the client app for the server app role.

- SSL errors: Don’t override cert store (REQUESTS_CA_BUNDLE, SSL_CERT_FILE) unless you know why.

- Tool not found / wrong schema: Recheck tool registration name & schema in server.py, rebuild, and redeploy.

- LangChain tool wrapper: When auto-wrapping MCP tools, make sure each generated function has a docstring; LangChain uses it as the tool description.

### Why this approach

- Security: Entra ID + app roles keeps tools behind enterprise auth.

- Operability: ACA revisions provide blue/green style rollouts.

- Dev velocity: Add tools in server.py, run one script, test, and go live—without stopping production.

### Get the code

- Server, examples, infra: https://github.com/PrynAI/PrynAI-MCP

#### If you ship your own org tools behind MCP, tag me—I’d love to see it.
