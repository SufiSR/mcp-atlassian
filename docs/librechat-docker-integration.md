# LibreChat + mcp-atlassian: Docker Integration Analysis

> Analysis date: 2026-03-13  
> Scope: Feasibility, required changes, credential caching, and proposed LibreChat YAML config

---

## Table of Contents

1. [TL;DR](#tldr)
2. [Is it currently possible?](#is-it-currently-possible)
3. [How the server handles per-request auth today](#how-the-server-handles-per-request-auth-today)
4. [The LibreChat integration challenge](#the-librechat-integration-challenge)
5. [What code changes are needed](#what-code-changes-are-needed)
6. [Credential caching analysis](#credential-caching-analysis)
7. [Docker deployment architecture for two environments](#docker-deployment-architecture-for-two-environments)
8. [Proposed librechat.yaml configuration](#proposed-librechatyaml-configuration)

---

## TL;DR

| Question | Answer |
|---|---|
| Is `streamable-http` transport supported? | **Yes** — fully implemented |
| Is per-request user auth via headers supported? | **Yes** — via `Authorization: Basic/Bearer/Token` |
| Does it work out-of-the-box with LibreChat's separate email + token vars? | **No** — Basic auth requires a pre-encoded base64 string |
| Are there credential caching/leakage issues? | **No** — fetchers are per-request, fully isolated |
| Do you need two separate containers for two environments? | **Yes** — `JIRA_URL` / `CONFLUENCE_URL` are startup-time globals |
| What code change is needed for clean LibreChat UX? | Add `X-Atlassian-Email` + `X-Atlassian-Token` header support in `UserTokenMiddleware` |

---

## Is it currently possible?

### Transport — Yes

The CLI and server already support `streamable-http`:

```
TRANSPORT=streamable-http   # or --transport streamable-http
PORT=8000
HOST=0.0.0.0
```

The server exposes the MCP endpoint at `/mcp` (configurable via `STREAMABLE_HTTP_PATH`) and a health check at `/healthz`. LibreChat's `streamable-http` type maps directly to this.

### Per-request authentication — Partially

The `UserTokenMiddleware` in `src/mcp_atlassian/servers/main.py` already processes these headers on every POST to `/mcp`:

| Header | Auth type activated |
|---|---|
| `Authorization: Basic <base64(email:token)>` | Basic auth (Cloud or Server/DC) |
| `Authorization: Bearer <token>` | OAuth access token |
| `Authorization: Token <pat>` | Server/DC PAT |
| `X-Atlassian-Jira-Personal-Token` + `X-Atlassian-Jira-Url` | Jira PAT (header-only mode, no global config needed) |
| `X-Atlassian-Confluence-Personal-Token` + `X-Atlassian-Confluence-Url` | Confluence PAT (header-only mode) |

The middleware stores extracted credentials on `request.state` and a fresh fetcher instance is created per request in `src/mcp_atlassian/servers/dependencies.py`.

**Conclusion: The server is architecturally ready for LibreChat.** The gap is purely at the HTTP header interface level: LibreChat can pass two separate variables (`{{MY_EMAIL}}` and `{{MY_API_TOKEN}}`), but the server currently only accepts them pre-combined as `Authorization: Basic <base64(email:token)>`.

---

## How the server handles per-request auth today

### Startup (global config)

`JIRA_URL` (and optionally `JIRA_USERNAME` / `JIRA_API_TOKEN`) are read once during server startup via `JiraConfig.from_env()`. This global config is frozen in `MainAppContext` and **never mutated** across the server's lifetime.

The global config is required at startup for the basic-auth and OAuth/PAT-Bearer branches (it provides the base URL and SSL/proxy settings). It is **not** required only when using the pure header-based PAT branch (`X-Atlassian-Jira-Url` + `X-Atlassian-Jira-Personal-Token`), which can operate without any env-var config if `ATLASSIAN_OAUTH_ENABLE=true` is set.

### Per-request (user-specific fetchers)

On each incoming POST, `_get_fetcher()` in `dependencies.py` follows this logic:

1. **Branch 1 — Header-based PAT** (`X-Atlassian-*-Url` + `X-Atlassian-*-Personal-Token` headers, no `Authorization`):  
   Builds a brand-new `JiraConfig` / `ConfluenceConfig` entirely from the headers. No global config needed.

2. **Branch 2 — Basic auth** (`Authorization: Basic <base64>`):  
   Clones the global config via `dataclasses.replace()`, overriding only `username` and `api_token` with the request's decoded credentials. URL and SSL settings come from the global config.

3. **Branch 3 — OAuth/PAT Bearer** (`Authorization: Bearer` or `Authorization: Token`):  
   Clones the global config, overriding only the token. URL and cloud_id come from the global config.

4. **Global fallback**: If no user headers are present, the server uses the global service-account credentials.

---

## The LibreChat integration challenge

### What LibreChat sends

LibreChat's `customUserVars` allows users to fill in named values which are then interpolated into request headers:

```yaml
headers:
  Authorization: "Basic {{ATLASSIAN_BASIC_AUTH}}"
customUserVars:
  ATLASSIAN_BASIC_AUTH:
    title: "Atlassian Credentials (base64)"
    description: "Enter base64(email:api_token)"
```

LibreChat **cannot** construct a base64 encoding from two separate variables. It only does string interpolation.

### The gap for Cloud basic auth

Atlassian Cloud uses email + API token for basic auth. The wire format is:

```
Authorization: Basic <base64("email@example.com:api_token_here")>
```

With LibreChat's two-variable UX (`{{MY_EMAIL}}` and `{{MY_API_TOKEN}}`), the server would need to accept these as two separate headers and construct the Basic auth internally. **This is not currently implemented.**

### What works today without any code changes

#### Option A — Single base64-encoded credential (poor UX, works for Cloud)

The user is asked to enter a pre-encoded value:

```yaml
headers:
  Authorization: "Basic {{ATLASSIAN_BASIC_AUTH}}"
customUserVars:
  ATLASSIAN_BASIC_AUTH:
    title: "Atlassian Basic Auth Token"
    description: >
      Enter your base64-encoded credentials: base64(email:api_token).
      You can generate this at https://www.base64encode.org/ using the format
      'your@email.com:your_api_token'
```

This works immediately but puts the burden of base64-encoding on the user.

#### Option B — PAT header mode (clean UX, Server/DC only)

For Server/DC instances, users have Personal Access Tokens (single strings). This maps cleanly to LibreChat's single-variable approach, and the URL can be provided as a second variable:

```yaml
headers:
  X-Atlassian-Jira-Personal-Token: "{{JIRA_PAT}}"
  X-Atlassian-Jira-Url: "{{JIRA_URL}}"
  X-Atlassian-Confluence-Personal-Token: "{{CONFLUENCE_PAT}}"
  X-Atlassian-Confluence-Url: "{{JIRA_URL}}/wiki"
customUserVars:
  JIRA_PAT:
    title: "Jira Personal Access Token"
    description: "Your Jira PAT from Profile → Personal Access Tokens"
  JIRA_URL:
    title: "Jira Instance URL"
    description: "e.g. https://jira.your-company.com"
```

The server must be started with `ATLASSIAN_OAUTH_ENABLE=true` instead of `JIRA_URL` for this mode.

**Note**: `ATLASSIAN_OAUTH_ENABLE=true` is intended for OAuth BYOT mode but its side-effect of suppressing the `JIRA_URL` requirement at startup makes it usable to enable pure header-driven PAT mode. No OAuth flow is actually used.

---

## What code changes are needed

For the desired clean UX (separate `{{MY_EMAIL}}` + `{{MY_API_TOKEN}}` LibreChat variables with Cloud basic auth), two separate headers need to be supported:

```
X-Atlassian-Email: user@example.com
X-Atlassian-Token: user_api_token
```

### Files to change

#### 1. `src/mcp_atlassian/servers/main.py` — `_process_authentication_headers()`

In `UserTokenMiddleware._process_authentication_headers()` (around line 484), after extracting service headers, add detection for the new headers:

```python
email_header = headers.get(b"x-atlassian-email")
token_header = headers.get(b"x-atlassian-token")

if email_header and token_header:
    email_str = email_header.decode("latin-1").strip()
    token_str = token_header.decode("latin-1").strip()
    if email_str and token_str:
        scope["state"]["user_atlassian_email"] = email_str
        scope["state"]["user_atlassian_api_token"] = token_str
        scope["state"]["user_atlassian_auth_type"] = "basic"
        scope["state"]["user_atlassian_token"] = None
```

This would allow LibreChat to pass:
```yaml
headers:
  X-Atlassian-Email: "{{MY_EMAIL}}"
  X-Atlassian-Token: "{{MY_API_TOKEN}}"
```

#### 2. No changes needed to `dependencies.py`

The basic auth branch in `_get_fetcher()` already reads `request.state.user_atlassian_email` and `request.state.user_atlassian_api_token` — it does not care how those values got into `request.state`.

#### 3. Startup config — `JIRA_URL` must still be set

The `JIRA_URL` (and `CONFLUENCE_URL`) must be set at container startup when using basic auth via headers, because `_create_user_config_for_fetcher()` clones the global config to inherit URL and SSL settings. The URL cannot come from the per-request headers in basic auth mode.

---

## Credential caching analysis

### No cross-request leakage

| Component | Scope | Cached? |
|---|---|---|
| `MainAppContext` | Process lifetime, frozen dataclass | Yes — immutable, intentional |
| `JiraConfig` / `ConfluenceConfig` (user-specific) | Per-request, via `dataclasses.replace()` | No — created fresh each request |
| `JiraFetcher` / `ConfluenceFetcher` (user-specific) | Per-request, stored on `request.state` | Yes — but scoped to single HTTP request |
| `TTLCache` at module level in `main.py` | Defined but **unused** | N/A |

The per-request fetcher is stored on `request.state` (a Starlette `State` object that is created fresh per HTTP connection). This is completely isolated between users and requests.

### Global config is never mutated

All three auth branches use `dataclasses.replace(global_config, ...)` to produce a new config object — the original is never modified. The `frozen=True` setting on `MainAppContext` enforces this at the Python level.

### OAuth token store (not applicable here)

When using full OAuth (not this use case), `OAuthConfig._save_tokens()` writes to the system keyring and `~/.mcp-atlassian/oauth-*.json`. This is a global per-process store, but since you are not using OAuth, this does not apply.

### Verdict: No credential caching issues

Each user's request creates its own fetcher with its own credentials, scoped to that request's lifetime. There is no risk of User A's token being visible to User B's request.

---

## Docker deployment architecture for two environments

Since `JIRA_URL` and `CONFLUENCE_URL` are startup-time environment variables, **each Atlassian environment needs its own container**.

```
LibreChat
  ├─ MCP Server: mcp-atlassian-env1  (container, JIRA_URL=https://env1.atlassian.net)
  └─ MCP Server: mcp-atlassian-env2  (container, JIRA_URL=https://env2.atlassian.net)
```

### docker-compose.yml

```yaml
services:
  mcp-atlassian-env1:
    build: .
    image: mcp-atlassian:local
    ports:
      - "8001:8000"
    environment:
      TRANSPORT: streamable-http
      PORT: "8000"
      HOST: "0.0.0.0"
      JIRA_URL: "https://env1.atlassian.net"
      CONFLUENCE_URL: "https://env1.atlassian.net/wiki"
      # Provide startup credentials as a service-account fallback
      # OR omit and rely on per-request Basic auth headers only
      # (per-request headers override these anyway)
      JIRA_USERNAME: "service-account@env1.com"
      JIRA_API_TOKEN: "service_account_api_token_env1"
      CONFLUENCE_USERNAME: "service-account@env1.com"
      CONFLUENCE_API_TOKEN: "service_account_api_token_env1"
      MCP_VERBOSE: "true"
    restart: unless-stopped

  mcp-atlassian-env2:
    build: .
    image: mcp-atlassian:local
    ports:
      - "8002:8000"
    environment:
      TRANSPORT: streamable-http
      PORT: "8000"
      HOST: "0.0.0.0"
      JIRA_URL: "https://env2.atlassian.net"
      CONFLUENCE_URL: "https://env2.atlassian.net/wiki"
      JIRA_USERNAME: "service-account@env2.com"
      JIRA_API_TOKEN: "service_account_api_token_env2"
      CONFLUENCE_USERNAME: "service-account@env2.com"
      CONFLUENCE_API_TOKEN: "service_account_api_token_env2"
      MCP_VERBOSE: "true"
    restart: unless-stopped
```

> **Why startup credentials?** Setting `JIRA_USERNAME` / `JIRA_API_TOKEN` at startup is optional if you always send per-request auth headers. However, providing them serves two purposes:  
> 1. They allow the server to start and validate config successfully  
> 2. They act as a service-account fallback if a user's headers are absent or malformed

---

## Proposed librechat.yaml configuration

Below are two variants depending on whether code changes have been made.

### Variant A — Without code changes (base64 workaround)

Users must enter their pre-encoded `base64(email:api_token)` string. Works today.

```yaml
mcpServers:
  atlassian-env1:
    type: streamable-http
    url: "http://mcp-atlassian-env1:8000/mcp"
    # If LibreChat is not in the same Docker network, use the mapped port:
    # url: "http://localhost:8001/mcp"
    headers:
      Authorization: "Basic {{ATLASSIAN_ENV1_BASIC_AUTH}}"
    customUserVars:
      ATLASSIAN_ENV1_BASIC_AUTH:
        title: "Env1 Atlassian Credentials"
        description: >
          Enter your base64-encoded credentials for the Env1 Atlassian instance.
          Format: base64(your@email.com:your_api_token).
          Generate it at <a href='https://www.base64encode.org/' target='_blank'>base64encode.org</a>
          using the format 'email:api_token' (no quotes).

  atlassian-env2:
    type: streamable-http
    url: "http://mcp-atlassian-env2:8000/mcp"
    headers:
      Authorization: "Basic {{ATLASSIAN_ENV2_BASIC_AUTH}}"
    customUserVars:
      ATLASSIAN_ENV2_BASIC_AUTH:
        title: "Env2 Atlassian Credentials"
        description: >
          Enter your base64-encoded credentials for the Env2 Atlassian instance.
          Format: base64(your@email.com:your_api_token).
          Generate it at <a href='https://www.base64encode.org/' target='_blank'>base64encode.org</a>
          using the format 'email:api_token' (no quotes).
```

### Variant B — After implementing separate email + token header support

After adding `X-Atlassian-Email` / `X-Atlassian-Token` header support in `UserTokenMiddleware` (see [code changes section](#what-code-changes-are-needed)):

```yaml
mcpServers:
  atlassian-env1:
    type: streamable-http
    url: "http://mcp-atlassian-env1:8000/mcp"
    headers:
      X-Atlassian-Email: "{{ATLASSIAN_ENV1_EMAIL}}"
      X-Atlassian-Token: "{{ATLASSIAN_ENV1_TOKEN}}"
    customUserVars:
      ATLASSIAN_ENV1_EMAIL:
        title: "Env1 Email"
        description: "Your Atlassian account email for the Env1 instance"
      ATLASSIAN_ENV1_TOKEN:
        title: "Env1 API Token"
        description: >
          Your Atlassian API token for the Env1 instance.
          Create one at <a href='https://id.atlassian.com/manage-profile/security/api-tokens' target='_blank'>id.atlassian.com</a>

  atlassian-env2:
    type: streamable-http
    url: "http://mcp-atlassian-env2:8000/mcp"
    headers:
      X-Atlassian-Email: "{{ATLASSIAN_ENV2_EMAIL}}"
      X-Atlassian-Token: "{{ATLASSIAN_ENV2_TOKEN}}"
    customUserVars:
      ATLASSIAN_ENV2_EMAIL:
        title: "Env2 Email"
        description: "Your Atlassian account email for the Env2 instance"
      ATLASSIAN_ENV2_TOKEN:
        title: "Env2 API Token"
        description: >
          Your Atlassian API token for the Env2 instance.
          Create one at <a href='https://id.atlassian.com/manage-profile/security/api-tokens' target='_blank'>id.atlassian.com</a>
```

### Variant C — Server/DC with PAT (works today, no code changes)

If your environments are Jira Server/Data Center (not Cloud), PAT mode works cleanly with LibreChat today. In this case the URL can also come from the header, enabling a single container for multiple environments if desired.

Container startup (only one container needed — URL comes from headers):

```yaml
# docker-compose.yml entry
mcp-atlassian-serverdc:
  build: .
  image: mcp-atlassian:local
  ports:
    - "8001:8000"
  environment:
    TRANSPORT: streamable-http
    PORT: "8000"
    HOST: "0.0.0.0"
    ATLASSIAN_OAUTH_ENABLE: "true"   # suppresses requirement for JIRA_URL at startup
    MCP_VERBOSE: "true"
```

LibreChat config:

```yaml
mcpServers:
  atlassian-env1-dc:
    type: streamable-http
    url: "http://mcp-atlassian-serverdc:8000/mcp"
    headers:
      X-Atlassian-Jira-Url: "https://jira.env1.example.com"
      X-Atlassian-Jira-Personal-Token: "{{JIRA_ENV1_PAT}}"
      X-Atlassian-Confluence-Url: "https://confluence.env1.example.com"
      X-Atlassian-Confluence-Personal-Token: "{{CONFLUENCE_ENV1_PAT}}"
    customUserVars:
      JIRA_ENV1_PAT:
        title: "Jira Env1 Personal Access Token"
        description: "Your PAT from Jira Profile → Personal Access Tokens"
      CONFLUENCE_ENV1_PAT:
        title: "Confluence Env1 Personal Access Token"
        description: "Your PAT from Confluence Profile → Personal Access Tokens"

  atlassian-env2-dc:
    type: streamable-http
    url: "http://mcp-atlassian-serverdc:8000/mcp"
    headers:
      X-Atlassian-Jira-Url: "https://jira.env2.example.com"
      X-Atlassian-Jira-Personal-Token: "{{JIRA_ENV2_PAT}}"
      X-Atlassian-Confluence-Url: "https://confluence.env2.example.com"
      X-Atlassian-Confluence-Personal-Token: "{{CONFLUENCE_ENV2_PAT}}"
    customUserVars:
      JIRA_ENV2_PAT:
        title: "Jira Env2 Personal Access Token"
        description: "Your PAT from Jira Profile → Personal Access Tokens"
      CONFLUENCE_ENV2_PAT:
        title: "Confluence Env2 Personal Access Token"
        description: "Your PAT from Confluence Profile → Personal Access Tokens"
```

> **Note for Variant C**: With `ATLASSIAN_OAUTH_ENABLE=true` and no global `JIRA_URL`, the tool list shown to the user before they provide headers may be empty (tools are hidden when no service config is available). Tools become visible once the headers are present in requests. This is a known behavior of the header-based service availability detection logic in `_list_tools_mcp()`.

---

## Summary of recommendations

| Scenario | Recommended approach | Code changes needed? |
|---|---|---|
| Atlassian Cloud, best UX | Add `X-Atlassian-Email` + `X-Atlassian-Token` header support | **Yes** (small, ~10 lines in middleware) |
| Atlassian Cloud, no code changes | Use `Authorization: Basic <base64>` single variable in LibreChat | No |
| Jira Server/Data Center | Use `X-Atlassian-Jira-Personal-Token` header (PAT mode) | No |
| Two Cloud environments | Two separate Docker containers, each with its own `JIRA_URL` | No |
| Two Server/DC environments | One or two containers; URLs can come from headers in PAT mode | No |

The minimal, highest-value code change is implementing `X-Atlassian-Email` + `X-Atlassian-Token` header parsing in `UserTokenMiddleware._process_authentication_headers()` in `src/mcp_atlassian/servers/main.py`. This unblocks clean LibreChat integration for Atlassian Cloud users without touching any other part of the codebase.
