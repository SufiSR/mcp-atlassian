# LibreChat + mcp-atlassian: Atlassian Cloud Setup Guide

This guide walks through running **mcp-atlassian** as a Docker container and connecting it to **LibreChat** using per-user Atlassian Cloud credentials (email + API token).

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [How authentication works](#how-authentication-works)
3. [Step 1 — Build or pull the Docker image](#step-1--build-or-pull-the-docker-image)
4. [Step 2 — Configure environment variables](#step-2--configure-environment-variables)
5. [Step 3 — Start the containers](#step-3--start-the-containers)
6. [Step 4 — Configure LibreChat](#step-4--configure-librechat)
7. [Step 5 — Users enter their credentials in LibreChat](#step-5--users-enter-their-credentials-in-librechat)
8. [Running two Atlassian environments](#running-two-atlassian-environments)
9. [Verifying the setup](#verifying-the-setup)
10. [Troubleshooting](#troubleshooting)
11. [Security notes](#security-notes)

---

## Prerequisites

| Requirement | Details |
|---|---|
| Docker + Docker Compose | v2.x recommended |
| LibreChat | With MCP server support and `customUserVars` enabled |
| Atlassian Cloud account | Each user needs their own API token |
| API tokens | Created at [id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens) |

---

## How authentication works

```
User in LibreChat
    │  enters email + API token once (stored encrypted by LibreChat)
    │
    ▼
LibreChat MCP client
    │  adds headers to every request:
    │    X-Atlassian-Email: user@company.com
    │    X-Atlassian-Token: <api_token>
    │
    ▼
mcp-atlassian container  (streamable-http, port 8000)
    │  UserTokenMiddleware extracts headers → sets basic-auth state
    │  _get_fetcher() clones global config, replaces credentials
    │  creates a fresh JiraFetcher / ConfluenceFetcher per request
    │
    ▼
Atlassian Cloud API  (https://your-instance.atlassian.net)
    │  receives standard HTTP Basic auth
    │  authenticated as the individual user, not a service account
```

Each user's credentials are used only for that user's requests. There is no credential sharing or caching between users.

The container still requires `JIRA_URL` and `CONFLUENCE_URL` at startup (to provide the base URL for all requests), plus optional service-account credentials as a fallback. Per-user headers always override the service-account credentials.

---

## Step 1 — Build or pull the Docker image

This fork contains a custom extension (the `X-Atlassian-Email` / `X-Atlassian-Token`
headers) that is not in the upstream published image. You must build the image from
source.

```bash
git clone https://github.com/SufiSR/mcp-atlassian.git
cd mcp-atlassian
docker build -t mcp-atlassian:local .
```

The `docker-compose.librechat.yml` already uses `build: .` by default, so no further
changes are needed after cloning.

---

## Step 2 — Configure environment variables

Copy the template and fill in your values:

```bash
cp .env.librechat.example .env.librechat
```

Edit `.env.librechat`:

```dotenv
# ── Environment 1 ──────────────────────────────────────────────────────────────

# Required: your Atlassian Cloud instance URL (no trailing slash)
JIRA_URL_ENV1=https://your-company.atlassian.net
CONFLUENCE_URL_ENV1=https://your-company.atlassian.net/wiki

# Optional: service-account credentials used as a fallback when no per-user
# headers are present. If you only want per-user auth, you can leave these
# blank — but the container will log a warning on startup.
JIRA_SERVICE_ACCOUNT_EMAIL_ENV1=mcp-bot@your-company.com
JIRA_SERVICE_ACCOUNT_TOKEN_ENV1=service_account_api_token_here
CONFLUENCE_SERVICE_ACCOUNT_EMAIL_ENV1=mcp-bot@your-company.com
CONFLUENCE_SERVICE_ACCOUNT_TOKEN_ENV1=service_account_api_token_here

# ── Environment 2 (if you have a second Atlassian instance) ───────────────────
JIRA_URL_ENV2=https://your-other-company.atlassian.net
CONFLUENCE_URL_ENV2=https://your-other-company.atlassian.net/wiki
JIRA_SERVICE_ACCOUNT_EMAIL_ENV2=mcp-bot@other-company.com
JIRA_SERVICE_ACCOUNT_TOKEN_ENV2=service_account_api_token_here
CONFLUENCE_SERVICE_ACCOUNT_EMAIL_ENV2=mcp-bot@other-company.com
CONFLUENCE_SERVICE_ACCOUNT_TOKEN_ENV2=service_account_api_token_here
```

> **API tokens**: Each user creates their own at  
> [id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens)  
> The service account token above is only for the optional fallback.

---

## Step 3 — Start the containers

```bash
docker compose -f docker-compose.librechat.yml up -d
```

Verify both containers are healthy:

```bash
docker compose -f docker-compose.librechat.yml ps
curl http://localhost:8001/healthz   # env1
curl http://localhost:8002/healthz   # env2
```

Both should return `{"status": "ok"}`.

---

## Step 4 — Configure LibreChat

Add the following to your LibreChat configuration file (typically `librechat.yaml`).

> **Important**: Replace the URL hostnames below with the actual hostname or IP
> that LibreChat's backend can reach the containers on. If LibreChat and the
> mcp-atlassian containers are in the **same Docker network**, use the container
> service name (e.g. `mcp-atlassian-env1`). If they are on separate hosts, use
> the host IP and the mapped port (`8001` / `8002`).

```yaml
mcpServers:

  # ── Atlassian Environment 1 ────────────────────────────────────────────────
  atlassian-env1:
    type: streamable-http
    # Same Docker network:
    url: "http://mcp-atlassian-env1:8000/mcp"
    # Different host / exposed port:
    # url: "http://<host-ip>:8001/mcp"
    headers:
      X-Atlassian-Email: "{{ATLASSIAN_ENV1_EMAIL}}"
      X-Atlassian-Token: "{{ATLASSIAN_ENV1_TOKEN}}"
    customUserVars:
      ATLASSIAN_ENV1_EMAIL:
        title: "Env1 — Atlassian Email"
        description: "Your Atlassian account email address for the primary instance (e.g. you@company.com)"
      ATLASSIAN_ENV1_TOKEN:
        title: "Env1 — Atlassian API Token"
        description: >
          Your personal Atlassian API token for the primary instance.
          Create one at <a href='https://id.atlassian.com/manage-profile/security/api-tokens'
          target='_blank'>id.atlassian.com</a>

  # ── Atlassian Environment 2 ────────────────────────────────────────────────
  atlassian-env2:
    type: streamable-http
    url: "http://mcp-atlassian-env2:8000/mcp"
    # url: "http://<host-ip>:8002/mcp"
    headers:
      X-Atlassian-Email: "{{ATLASSIAN_ENV2_EMAIL}}"
      X-Atlassian-Token: "{{ATLASSIAN_ENV2_TOKEN}}"
    customUserVars:
      ATLASSIAN_ENV2_EMAIL:
        title: "Env2 — Atlassian Email"
        description: "Your Atlassian account email address for the secondary instance"
      ATLASSIAN_ENV2_TOKEN:
        title: "Env2 — Atlassian API Token"
        description: >
          Your personal Atlassian API token for the secondary instance.
          Create one at <a href='https://id.atlassian.com/manage-profile/security/api-tokens'
          target='_blank'>id.atlassian.com</a>
```

After editing, restart LibreChat to apply the changes.

---

## Step 5 — Users enter their credentials in LibreChat

The first time a user activates one of the Atlassian MCP servers in LibreChat, they will be prompted to fill in:

1. **Atlassian Email** — their login email for that Atlassian instance
2. **Atlassian API Token** — a personal token they generate at Atlassian's token page

LibreChat stores these values encrypted per-user. They are sent as plain HTTP headers (`X-Atlassian-Email` and `X-Atlassian-Token`) to the mcp-atlassian container on every request.

### Generating an API token

1. Go to [id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens)
2. Click **Create API token**
3. Give it a label (e.g. "LibreChat MCP")
4. Copy the token — it is shown only once
5. Paste it into the LibreChat credential prompt

---

## Running two Atlassian environments

If you only need one environment, remove the `atlassian-env2` service from `docker-compose.librechat.yml` and the corresponding `atlassian-env2` block from `librechat.yaml`.

For **more than two** environments, duplicate and rename the service blocks, incrementing the port number each time.

---

## Verifying the setup

### Check container logs

```bash
# Watch env1 logs in real time
docker compose -f docker-compose.librechat.yml logs -f mcp-atlassian-env1
```

A successful per-user request will show lines like:

```
Creating user-specific JiraFetcher (type: basic) for user user@company.com
get_jira_fetcher: Validated Jira basic auth for user ID: <account-id>
```

### Test with curl

```bash
# Replace with your values
curl -X POST http://localhost:8001/mcp \
  -H "Content-Type: application/json" \
  -H "X-Atlassian-Email: you@company.com" \
  -H "X-Atlassian-Token: your_api_token" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

A successful response returns a JSON object with a `result.tools` array listing all available Jira and Confluence tools.

### Test the health endpoint

```bash
curl http://localhost:8001/healthz
# Expected: {"status": "ok"}
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401 Unauthorized: X-Atlassian-Email header is empty` | LibreChat sent an empty email | User needs to re-enter credentials in LibreChat |
| `401 Unauthorized: X-Atlassian-Token header is empty` | LibreChat sent an empty token | User needs to re-enter credentials in LibreChat |
| `Invalid user Jira token or configuration` | Wrong email or expired token | User should regenerate their API token at Atlassian |
| Container starts but Jira tools are hidden | `JIRA_URL` is not set or invalid | Check `JIRA_URL_ENV1` in `.env.librechat` |
| `Connection refused` from LibreChat | Wrong URL / network | Verify container name and network in `librechat.yaml` |
| `403 Forbidden: Invalid Jira URL` | URL in header is blocked by SSRF check | Only `JIRA_URL` from env is used for basic auth — this should not happen |

### Enable verbose logging

Add to the container's environment:

```dotenv
MCP_VERY_VERBOSE=true
```

Then restart and check logs:

```bash
docker compose -f docker-compose.librechat.yml up -d
docker compose -f docker-compose.librechat.yml logs -f
```

---

## Security notes

- **Headers are plaintext over the internal network**. Ensure mcp-atlassian containers are not exposed to the public internet — only LibreChat's backend should reach them. Use Docker networks or a firewall.
- **API tokens have the same permissions as the user**. Rotate them regularly.
- **Service-account credentials** in `.env.librechat` are used only as a fallback when no user headers are present. Consider omitting them if every user always provides their own credentials.
- **`IGNORE_HEADER_AUTH=true`** disables all header-based auth. Do not set this in this deployment pattern.
- **Read-only mode**: Add `READ_ONLY_MODE=true` to the container environment to prevent any write operations regardless of user credentials.
