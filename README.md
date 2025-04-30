# n8n + MCP Quick‑Start Guide

A concise but complete walkthrough for standing up an **n8n workflow‑automation server** that can call any **Model Context Protocol (MCP) server** through **SuperGateway**, plus daily usage and restart instructions.

---

## 🗺 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Layout](#project-layout)
3. [One‑Time Setup](#one-time-setup)\
      3.1 [Create the folder](#31-create-the-folder)\
      3.2 [Write ](#32-write-docker-composeyml)[`docker-compose.yml`](#32-write-docker-composeyml)\
      3.3 [Create data volumes](#33-create-data-volumes)\
      3.4 [Start the stack](#34-start-the-stack)\
      3.5 [Verify running containers](#35-verify-running-containers)\
      3.6 [Test the Fetch endpoint](#36-test-the-fetch-endpoint)
4. [Configuring n8n](#configuring-n8n)\
      4.1 [First‑time login](#41-first-time-login)\
      4.2 [Create a workflow](#42-create-a-workflow)\
      4.3 [Add a Manual Trigger](#43-add-a-manual-trigger)\
      4.4 [Add an AI Agent node](#44-add-an-ai-agent-node)\
      4.5 [Attach an MCP Client Tool](#45-attach-an-mcp-client-tool)\
      4.6 [Create OpenRouter credentials](#46-create-openrouter-credentials)\
      4.7 [Attach an OpenRouter Chat Model](#47-attach-an-openrouter-chat-model)\
      4.8 [Save & Test](#48-save--test)
5. [Daily Usage / Restart](#daily-usage--restart)
6. [Troubleshooting](#troubleshooting)
7. [Extending the Stack](#extending-the-stack)

---

## Prerequisites

| Tool                   | Version         | Notes                                                     |
| ---------------------- | --------------- | --------------------------------------------------------- |
| **Docker Engine**      |  20.10 or newer | Docker Desktop on macOS/Windows works fine.               |
| **Docker Compose v2**  | Built‑in        | Part of modern Docker Desktop or `docker‑compose-plugin`. |
| **VS Code (optional)** | Any             | Only used for editing files with `code .`.                |

> **No local Node or Python installs are required**—everything runs in containers.

---

## Project Layout

```
~/Desktop/n8n-mcp/
├─ docker-compose.yml      # stack definition (n8n + SuperGateway‑wrapped MCPs)
├─ n8n_data/               # persistent SQLite DB for n8n
└─ fs_data/                # sample folder for the filesystem MCP
```

Feel free to create the folder anywhere; `~/Desktop` is used below for convenience.

---

## One‑Time Setup

### 3.1 Create the folder

```bash
mkdir -p ~/Desktop/n8n-mcp && cd ~/Desktop/n8n-mcp
```

(Optional) Open it in VS Code:

```bash
code .
```

### 3.2 Write `docker-compose.yml`

Paste the following **exactly**—indenting matters!\
`docker-compose.yml` ⤵

```yaml
version: "3.9"
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    ports:
      - "5678:5678"          # n8n UI
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - TZ=America/Los_Angeles
    volumes:
      - ./n8n_data:/home/node/.n8n
    networks: [n8nnet]
    depends_on: [mcp_fetch]

  # Example 1 – web‑scraping Fetch MCP server
  mcp_fetch:
    image: supercorp/supergateway:latest
    command: >
      --stdio "npx -y @kazuph/mcp-fetch"
      --port 8001
      --baseUrl http://mcp_fetch:8001
    ports:
      - "8001:8001"
    networks: [n8nnet]

  # Example 2 – local filesystem MCP (optional)
  mcp_fs:
    image: supercorp/supergateway:latest
    command: >
      --stdio "npx -y @modelcontextprotocol/server-filesystem /data"
      --port 8002
      --baseUrl http://mcp_fs:8002
    volumes:
      - ./fs_data:/data
    ports:
      - "8002:8002"
    networks: [n8nnet]

networks:
  n8nnet:
```

### 3.3 Create data volumes

```bash
mkdir -p n8n_data fs_data
```

### 3.4 Start the stack

```bash
docker compose up -d
```

Docker pulls three images and starts the containers.

### 3.5 Verify running containers

```bash
docker compose ps
```

You should see **n8n**, **mcp\_fetch**, **mcp\_fs** all in `Up` status.

### 3.6 Test the Fetch endpoint

```bash
curl -v -H "Accept: text/event-stream" http://localhost:8001/sse --max-time 5
```

Look for `HTTP/1.1 200 OK` followed by an `event:` line, then Ctrl‑C.

---

## Configuring n8n

### 4.1 First‑time login

- Visit [http://localhost:5678](http://localhost:5678).
- Create the Owner account when prompted.

### 4.2 Create a workflow

- Click **Start from scratch** on the Workflows page.

### 4.3 Add a Manual Trigger

- Click the **+** in the blank canvas → search “Manual” → **Manual Trigger**.

### 4.4 Add an AI Agent node

- Hover the Manual Trigger → small **⊕** → search “AI Agent”.

### 4.5 Attach an MCP Client Tool

- Select the AI Agent node → *Tools* → **+ Add tool** → **MCP Client Tool**.
- **SSE Endpoint**: `http://mcp_fetch:8001/sse`\
  (Use the service name—not `localhost`—so it resolves inside Docker.)
- Leave Authentication = *None*.

### 4.6 Create OpenRouter credentials

1. **Overview → Credentials → + New credential → OpenAI API**.
2. Name → `OpenRouter`\
   API Key → `sk-or‑…`\
   Base URL → `https://openrouter.ai/api`
3. **Save**.

### 4.7 Attach an OpenRouter Chat Model

- Under the AI Agent node, click the **Chat Model** diamond → search “OpenRouter Chat Model”.
- Pick the `OpenRouter` credential and your desired model ID (e.g., `anthropic/claude-3.7-sonnet`).

### 4.8 Save & Test

1. **Save** the workflow (top‑right).
2. Click **Open chat** → send:
   ```
   Please fetch https://n8n.io and give me a 2‑sentence summary.
   ```
3. You should receive a live summary fetched via the MCP Fetch server.

---

## Daily Usage / Restart

```bash
cd ~/Desktop/n8n-mcp       # project root

docker compose up -d      # start (or restart) the whole stack

docker compose ps         # confirm containers are Up
```

Open [http://localhost:5678](http://localhost:5678) — all workflows, credentials, and MCP servers are ready.

### Common lifecycle commands

| Purpose                                 | Command                                                     |
| --------------------------------------- | ----------------------------------------------------------- |
| Gracefully stop containers              | `docker compose stop`                                       |
| Stop & remove containers (keep volumes) | `docker compose down`                                       |
| Wipe everything (fresh slate)           | `docker compose down -v`                                    |
| Pull new images, then restart           | `docker compose pull && docker compose up -d`               |
| Tail logs for one service               | `docker compose logs -f n8n`                                |
| Re‑create one service after editing YML | `docker compose up -d --force-recreate --no-deps mcp_fetch` |

---

## Troubleshooting

| Symptom                       | Fix                                                                                                           |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `` in `mcp_fetch` logs        | Ensure the `command:` line uses `npx -y @kazuph/mcp-fetch` (working package). Restart that service.           |
| ``** hangs with no response** | Use `curl -v .../sse` with `Accept: text/event-stream` or check that the container’s port mapping is correct. |
| Workflow can’t reach MCP      | Make sure you used the **service name** (`mcp_fetch`) in the SSE endpoint, not `localhost`.                   |
| OpenRouter errors             | Verify API key, Base URL, and model string; check your OpenRouter quota.                                      |

---

## Extending the Stack

- **Add more MCP tools**: copy the `mcp_fetch` block, change the service name, port, and `--stdio` package, then add another MCP Client Tool in the Agent.
- **Persist memory**: connect a Vector DB Memory node to the Agent.
- **Webhook chat UI**: replace the Manual Trigger with an *n8n Chat UI* or *Webhook* node for public access.
- **Production hardened**: Reverse‑proxy behind Caddy/Traefik, mount Postgres for n8n, store secrets via env vars.

---

> **Enjoy automated agents with live tool‑calling!**\
> If you improve this setup, PRs are welcome.

