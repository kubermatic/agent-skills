---
name: kdp-mcp
description: Interact with Kubermatic Developer Platform (KDP) using MCP server tools. Use this when the user wants to discover services, enable them, create or manage resources, or explore their platform environment. Requires mcp-k8s-kcp and kubernetes MCP servers.
allowed-tools:
  - "mcp__mcp-k8s-kcp__listKcpWorkspaceAvailableApis"
  - "mcp__mcp-k8s-kcp__listServiceCatalog"
  - "mcp__mcp-k8s-kcp__listServiceBindings"
  - "mcp__mcp-k8s-kcp__getServiceCatalogEntry"
  - "mcp__mcp-k8s-kcp__getServiceBinding"
  - "mcp__mcp-k8s-kcp__createServiceBinding"
  - "mcp__mcp-k8s-kcp__deleteServiceBinding"
  - "mcp__mcp-k8s-kcp__getKCPApiSchema"
  - "mcp__kubernetes__kubectl_apply"
  - "mcp__kubernetes__kubectl_get"
  - "mcp__kubernetes__kubectl_describe"
  - "mcp__kubernetes__kubectl_delete"
  - "mcp__kubernetes__kubectl_logs"
  - "mcp__kubernetes__list_api_resources"
---

# KDP with MCP Server

You're helping a developer work with their Kubermatic Developer Platform (KDP) environment. You have two sets of MCP tools available — use the right one depending on what the user needs.

## Setup

This skill needs two MCP servers in `.mcp.json`:

```json
{
  "mcpServers": {
    "mcp-k8s-kcp": {
      "type": "stdio",
      "command": "/path/to/mcp-k8s-go",
      "env": { "KUBECONFIG": "/path/to/kubeconfig" }
    },
    "kubernetes": {
      "type": "stdio",
      "command": "npx",
      "args": ["mcp-server-kubernetes"]
    }
  }
}
```

## Two layers, two tool sets

KDP has a platform layer and a resource layer. Pick the right tools:

**`mcp-k8s-kcp` tools — for platform operations:**

- `listServiceCatalog` — browse what services exist ("what services can I use?")
- `getServiceCatalogEntry` — details about a specific service before enabling it
- `createServiceBinding` — enable a service (needs `exportName` + `exportPath` from the catalog)
- `deleteServiceBinding` — disable a service
- `listServiceBindings` / `getServiceBinding` — check what's enabled and its status
- `listKcpWorkspaceAvailableApis` — list resource kinds you can create right now
- `getKCPApiSchema` — get the schema for a resource. **Always call this before creating anything.**

**`kubernetes` tools — for working with actual resources:**

- `kubectl_apply` — create or update a resource from a YAML manifest
- `kubectl_get` — list resources or get one by name
- `kubectl_describe` — detailed status, events, conditions
- `kubectl_delete` — remove a resource
- `kubectl_logs` — container logs for debugging
- `list_api_resources` — all registered API types

## How to do things

**Enable a service:**
1. `listServiceCatalog` to see what's out there
2. `getServiceCatalogEntry` to check what it provides
3. `createServiceBinding` with the export name and path from step 2
4. `listServiceBindings` to confirm it's active

**Create a resource (database, certificate, whatever):**
1. `listKcpWorkspaceAvailableApis` to confirm the resource type exists
2. `getKCPApiSchema` with the resource name — read the schema, note required fields
3. Build a YAML manifest based on that schema and what the user asked for
4. `kubectl_apply` to create it
5. `kubectl_get` to verify it exists

**Check status and get credentials:**
1. `kubectl_get` or `kubectl_describe` the resource
2. Look for `related-resources.kdp.k8c.io/*` annotations — these point to Secrets or ConfigMaps with connection info
3. Get those related resources to find credentials

## Rules

**Don't expose platform internals.** The user doesn't need to know about KCP, APIBindings, APIExports, or workspace paths. Use plain language:
- "enable a service" not "create an APIBinding"
- "the service catalog" not "APIExports"
- "your environment" not "your KCP workspace"
- "this service isn't available" not "APIExport not found"

**Always fetch the schema first.** Before creating any resource, call `getKCPApiSchema`. Don't guess field names or structure.

**Resources have custom API groups.** Things like `kkp.kdp.io`, `mongodb.com`, `databases.example.corp` are normal — they're platform-provided services.

**spec goes in, status comes back.** The user writes the `spec`. The service provider populates `status` with provisioning progress and connection details.

**Related resources carry credentials.** Services often create Secrets or ConfigMaps alongside the main resource. Check for `related-resources.kdp.k8c.io/*` annotations to find them.
