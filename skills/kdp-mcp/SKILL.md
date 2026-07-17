---
name: kdp-mcp
description: Interact with Kubermatic Developer Platform (KDP) using MCP server tools. Use this when the user wants to discover services, enable them, create or manage resources, or explore their platform environment. Requires mcp-kdp and kubernetes MCP servers.
allowed-tools:
  - "mcp__mcp-kdp__listKcpWorkspaceAvailableApis"
  - "mcp__mcp-kdp__listServiceCatalog"
  - "mcp__mcp-kdp__listServiceBindings"
  - "mcp__mcp-kdp__getServiceCatalogEntry"
  - "mcp__mcp-kdp__getServiceBinding"
  - "mcp__mcp-kdp__createServiceBinding"
  - "mcp__mcp-kdp__deleteServiceBinding"
  - "mcp__mcp-kdp__getKCPApiSchema"
  - "mcp__mcp-kdp__listBlueprintDefinitions"
  - "mcp__mcp-kdp__getBlueprintDefinition"
  - "mcp__mcp-kdp__createBlueprintDefinition"
  - "mcp__mcp-kdp__publishBlueprintDefinition"
  - "mcp__mcp-kdp__listWorkspaces"
  - "mcp__mcp-kdp__getWorkspaceInfo"
  - "mcp__mcp-kdp__useWorkspace"
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
    "mcp-kdp": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "-v", "/absolute/path/to/your/kubeconfig:/home/nonroot/.kube/config:ro",
        "-v", "/absolute/path/to/your/oidc-login/cache:/home/nonroot/.kube/cache/oidc-login:ro",
        "quay.io/kubermatic/mcp-kdp:latest"
      ]
    },
    "kubernetes": {
      "type": "stdio",
      "command": "npx",
      "args": ["mcp-server-kubernetes"]
    }
  }
}
```

Your kubeconfig must be configured to authenticate with your OIDC provider (e.g. via `kubelogin` / `oidc-login`). The oidc-login cache mount lets the container reuse your existing tokens so you don't have to re-authenticate in the browser.

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

**`mcp-k8s-kcp` tools — for blueprint management:**

- `listBlueprintDefinitions` — list all blueprint definitions, optionally filter by namespace. Shows name, version, phase, published and deprecated status.
- `getBlueprintDefinition` — get details about a specific blueprint definition (version, phase, published status, composed resource kinds, conditions)
- `createBlueprintDefinition` — create a new blueprint definition from a YAML or JSON manifest
- `publishBlueprintDefinition` — publish or unpublish a blueprint definition to make it available for tenants

**`mcp-k8s-kcp` tools — for workspace navigation:**

- `listWorkspaces` — list child workspaces in the current workspace (name, type, phase, URL)
- `getWorkspaceInfo` — get current workspace path and cluster host URL
- `useWorkspace` — switch to a different workspace by path (e.g. `root:org:project`)

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

**Manage blueprints:**
1. `listBlueprintDefinitions` to see existing blueprints
2. `getBlueprintDefinition` to inspect details, composed resources, and conditions
3. `createBlueprintDefinition` with a YAML/JSON manifest to create a new one
4. `publishBlueprintDefinition` to make it available for tenants

**Navigate workspaces:**
1. `getWorkspaceInfo` to see where you are now
2. `listWorkspaces` to discover child workspaces
3. `useWorkspace` to switch to a different workspace by path

**Check status and get credentials:**
1. `kubectl_get` or `kubectl_describe` the resource
2. Look for `related-resources.kdp.k8c.io/*` annotations — these point to Secrets or ConfigMaps with connection info
3. Tell the user which secret/configmap exists and what keys it contains. **Don't dump secret values into the conversation** — if the user needs a specific value, extract just that key and warn them it's sensitive.

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
