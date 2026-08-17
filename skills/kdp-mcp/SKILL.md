---
name: kdp-mcp
description: Interact with Kubermatic Developer Platform (KDP) using the mcp-kdp server. Use this when the user wants to discover services, inspect or create resources, or explore their platform environment through MCP tools instead of kubectl. Requires the kdp MCP server.
allowed-tools:
  - "mcp__kdp__listWorkspaces"
  - "mcp__kdp__getWorkspaceInfo"
  - "mcp__kdp__getServiceKubeconfig"
  - "mcp__kdp__listServiceCatalog"
  - "mcp__kdp__getServiceCatalogEntry"
  - "mcp__kdp__listAvailableApis"
  - "mcp__kdp__getApiSchema"
  - "mcp__kdp__listResources"
  - "mcp__kdp__getResource"
  - "mcp__kdp__createResource"
  - "mcp__kdp__updateResource"
  - "mcp__kdp__deleteResource"
---

# KDP with MCP Server

You're helping a developer work with their Kubermatic Developer Platform (KDP) environment through the
`mcp-kdp` server. It exposes 12 tools covering both the platform layer (what services exist, where you
are) and the resource layer (create and inspect actual resources) — you do not need a separate
Kubernetes MCP server or any shell commands.

The tool prefix follows the server name in `.mcp.json`. This skill assumes the server is named `kdp`,
so the tools are `mcp__kdp__*`. If you named it something else, adjust accordingly.

## Setup

The platform hosts the MCP server behind a gateway that authenticates you with your normal platform
account, so there is nothing to install and no kubeconfig to manage:

```bash
claude mcp add --transport http \
  --client-id kdp-mcp \
  --callback-port 19876 \
  kdp https://public.api.<your-kdp-domain>/mcp
```

Or commit it to the repository as `.mcp.json`:

```json
{
  "mcpServers": {
    "kdp": {
      "type": "http",
      "url": "https://public.api.<your-kdp-domain>/mcp",
      "oauth": { "clientId": "kdp-mcp", "callbackPort": 19876 }
    }
  }
}
```

Run `/mcp` and complete the browser login. Every call then acts as **you**, with exactly your
permissions.

Two things to leave alone:

- **Do not change the callback port.** The matching redirect URI is pre-registered with the identity
  provider, which exact-matches it, so any other port is rejected at login.
- **Do not pin OAuth scopes.** The gateway advertises the scopes it needs — including a dynamic scope
  that makes the platform accept the token — and Claude Code takes them from there. Setting
  `oauth.scopes` overrides that and breaks authorization.

The dashboard generates this snippet for your environment under **Connect an AI agent**, which is the
easiest way to get the correct URL.

## Every call needs a workspace path

There is **no** "current workspace" and no way to switch into one — the server is stateless. Every
tool takes a `workspace` argument, a path like `root:demo` or `root:org:team`.

- If the user hasn't told you which workspace, **ask**, or call `listWorkspaces` on a path you already
  know to discover children. Don't guess.
- `getWorkspaceInfo` confirms a path resolves and returns its cluster host.
- Reuse the same path for follow-up calls in a conversation rather than asking repeatedly.

## The tools

**Platform layer:**

- `listServiceCatalog` — what services exist ("what can I use?")
- `getServiceCatalogEntry` — details for one service: resource schemas, permission claims, conditions
- `listWorkspaces` — child workspaces of a given path (name, type, phase, URL)
- `getWorkspaceInfo` — workspace path and cluster host URL
- `getServiceKubeconfig` — a kubeconfig blob for a workspace's service endpoint, for the user to save
  and use with kubectl

**Resource layer:**

- `listAvailableApis` — resource kinds you can actually use in this workspace (internal platform APIs
  are filtered out)
- `getApiSchema` — the OpenAPI schema for a kind. **Always call this before creating or updating.**
- `listResources` / `getResource` — read resources, optionally by namespace or label selector
- `createResource` / `updateResource` — from a YAML or JSON manifest
- `deleteResource` — by kind and name

## How to do things

**See what's available:**
1. `listServiceCatalog` for the catalog
2. `listAvailableApis` for what is usable in this workspace right now
3. `getServiceCatalogEntry` for details on anything the user asks about

**Create a resource (database, certificate, whatever):**
1. `listAvailableApis` to confirm the kind is usable here
2. `getApiSchema` for that kind — read it and note required fields
3. Build a manifest from the schema and what the user asked for
4. `createResource`
5. `getResource` to confirm it exists

**Check status and get credentials:**
1. `getResource` for the resource
2. Look for `related-resources.kdp.k8c.io/*` annotations — they point at Secrets or ConfigMaps holding
   connection info
3. Name the secret and its keys. **Don't dump secret values into the conversation.** If the user needs
   one specific value, fetch just that key and tell them it is sensitive.

## What these tools cannot do

Be straight with the user instead of improvising:

- **Enabling or disabling a service.** There is no binding tool. Enabling a service means creating an
  APIBinding, which is a platform-internal object these tools deliberately don't manage — point the
  user at the dashboard or the `kdp-kubectl` skill.
- **Authoring or publishing blueprints.** Use the `kdp-blueprints` skill.
- **Logs, exec, describe.** There is no `kubectl logs` equivalent. `getResource` returns the full
  object including `status` and conditions, which covers most debugging; beyond that, the user needs
  kubectl.

## Rules

**Don't expose platform internals.** The user doesn't need to hear about kcp, APIBindings, or
APIExports. Use plain language:
- "the service catalog" not "APIExports"
- "your environment" not "your kcp workspace"
- "this service isn't available here" not "APIExport not found"

The one exception is the workspace path, which the user has to give you and will recognise.

**Always fetch the schema first.** Before creating or updating anything, call `getApiSchema`. Don't
guess field names or nesting.

**Resources have custom API groups.** `kkp.kdp.io`, `mongodb.com`, `databases.example.corp` are normal
— they are platform-provided services.

**spec goes in, status comes back.** The user writes the `spec`. The service provider fills `status`
with provisioning progress and connection details.

**You act as the user.** The token carries their identity and permissions, so a permission error is
real and worth reporting plainly, not something to work around.
