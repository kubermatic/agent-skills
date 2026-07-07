# Kubermatic Agent Skills

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for working with [Kubermatic Developer Platform (KDP)](https://docs.kubermatic.com/developer-platform/). Install one of these skills so Claude knows how to talk to your KDP environment — browse the service catalog, enable services, create resources, check status, etc.

## Skills

### kdp

For platform users who just want to get things done. Uses kubectl under the hood but hides all the internal terminology — no mention of APIBindings, APIExports, kcp, or workspaces. Claude handles the plumbing and talks to you in plain language.

**Requires:** `kubectl` pointing at a KDP workspace.

### kdp-mcp

Uses the [mcp-k8s-go](https://github.com/kubermatic-labs/mcp-k8s-go) MCP server together with a Kubernetes MCP server. Claude calls the MCP tools directly — no shell commands needed.

**Requires:** `mcp-k8s-kcp` and `kubernetes` MCP servers in your `.mcp.json`.

### kdp-kubectl

For service providers and platform engineers who want to see the internals. Same kubectl workflows as `kdp` but uses the actual KCP terminology (APIBindings, APIExports, etc.).

**Requires:** `kubectl` pointing at a KDP workspace.

### kdp-blueprints

For Blueprint **authors** who want to compose several KDP services into one reusable, publishable kind (e.g. a `PostgresPair` that provisions two databases, or a `WebappStack` of database + repo + app). Walks through authoring a `BlueprintDefinition` with a kro `ResourceGraphDefinition`, validating it, publishing it to the service catalog, and smoke-testing an instance. Uses real platform terminology and includes an RGD authoring reference plus a complete example.

**Requires:** `kubectl` pointing at a KDP workspace with `blueprints.kdp.k8c.io` and the services you want to compose already bound.

## Installation

Copy whichever skill you need into your Claude skills directory:

```bash
# project-level (just this repo)
mkdir -p .claude/skills/
cp -r skills/kdp .claude/skills/

# global (all projects)
mkdir -p ~/.claude/skills/
cp -r skills/kdp ~/.claude/skills/
```

Replace `kdp` with `kdp-mcp`, `kdp-kubectl`, or `kdp-blueprints` if you want one of the other variants.

## Setup

### kdp / kdp-kubectl

1. Point your kubeconfig at a KDP workspace
2. Check it works: `kubectl api-resources`
3. Optionally install the [kcp kubectl plugin](https://github.com/kcp-dev/kcp) for workspace navigation

### kdp-mcp

1. Build or install [mcp-k8s-go](https://github.com/kubermatic-labs/mcp-k8s-go)
2. Add to your `.mcp.json`:

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

## Usage

Just ask Claude what you need:

```
> What services are available?
> Enable the database service
> Create a PostgreSQL database called orders-db with 20Gi storage
> What's the status of my database?
> What credentials do I need to connect?
```

## Contributing

1. Fork this repo
2. Add or modify skills under `skills/`
3. Each skill needs a `SKILL.md` with YAML frontmatter and instructions
4. Open a PR
