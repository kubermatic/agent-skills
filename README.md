# Kubermatic Agent Skills

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for working with [Kubermatic Developer Platform (KDP)](https://docs.kubermatic.com/developer-platform/). Install one of these skills so Claude knows how to talk to your KDP environment — browse the service catalog, enable services, create resources, check status, etc.

## Skills

### kdp-mcp

Uses the [mcp-k8s-go](https://github.com/kubermatic-labs/mcp-k8s-go) MCP server together with a Kubernetes MCP server. Claude calls the MCP tools directly — no shell commands needed.

**Requires:** `mcp-k8s-kcp` and `kubernetes` MCP servers in your `.mcp.json`.

### kdp-kubectl

Uses plain `kubectl` commands. Same workflows, no MCP servers needed.

**Requires:** `kubectl` pointing at a KDP workspace.

## Installation

Copy whichever skill you need into your Claude skills directory:

```bash
# project-level (just this repo)
mkdir -p .claude/skills/
cp -r skills/kdp-mcp .claude/skills/
# or
cp -r skills/kdp-kubectl .claude/skills/

# global (all projects)
mkdir -p ~/.claude/skills/
cp -r skills/kdp-mcp ~/.claude/skills/
# or
cp -r skills/kdp-kubectl ~/.claude/skills/
```

## Setup

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

### kdp-kubectl

1. Point your kubeconfig at a KDP workspace
2. Check it works: `kubectl api-resources`
3. Optionally install the [kcp kubectl plugin](https://github.com/kcp-dev/kcp) for workspace navigation

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
