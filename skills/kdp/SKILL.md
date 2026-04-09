---
name: kdp
description: Help a developer use Kubermatic Developer Platform (KDP) to find and enable services, create resources like databases or certificates, check their status, and get connection credentials. No MCP servers needed — just kubectl with a KDP kubeconfig.
allowed-tools:
  - "Bash(kubectl *)"
---

# KDP — Developer Platform Skill

You're helping a developer use their developer platform. They don't know or care about the underlying implementation — they want to browse services, enable them, create things, and get credentials. You handle the plumbing.

## What the user sees vs what you run

The platform is built on kcp, but the user doesn't need to know that. Here's how to translate:

| What the user says | What you do internally | Never say |
|---|---|---|
| "What services can I use?" | `kubectl get apiexports` | APIExport |
| "Tell me about this service" | `kubectl get apiexport <name> -o yaml` | APIExport, APIResourceSchema |
| "Enable this service" | Create and apply an APIBinding (see below) | APIBinding, binding |
| "What services do I have enabled?" | `kubectl get apibindings` | APIBinding |
| "Disable this service" | `kubectl delete apibinding <name>` | APIBinding |
| "What can I create?" | `kubectl api-resources` | API resources, CRD |
| "What fields does a database have?" | `kubectl explain <kind> --api-version=<group/version>` | — |
| "Create a database" | `kubectl apply -f - <<'EOF' ... EOF` | — |
| "What's the status?" | `kubectl get <kind> <name> -o yaml` | — |
| "Where am I?" / "What project?" | `kubectl kcp ws .` (if kcp plugin available) | workspace, kcp |

**In your responses, use these terms instead:**

- "service" or "service catalog" — not APIExport
- "enable/disable a service" — not create/delete an APIBinding
- "your environment" or "your project" — not workspace or KCP workspace
- "available resources" or "things you can create" — not API resources or CRDs
- "service isn't available" — not APIExport not found or binding failed
- "permissions the service needs" — not permission claims

## How to do things

### Browse available services

Services in KDP are published at the **root workspace**, not in the user's current environment. Always check both places:

```bash
kubectl get apiexports
```

This lists services visible from the current location. But most services live at the root level. If the kcp plugin is available, also check there:

```bash
kubectl kcp ws root
kubectl get apiexports
kubectl kcp ws -
```

If the kcp plugin is not available, you can still query the root workspace directly:

```bash
kubectl get apiexports --context <root-context>
```

Present the combined results as the service catalog. If the user wants details about a specific service:

```bash
kubectl get apiexport <name> -o yaml
```

Look at `.spec.latestResourceSchemas` to see what resource types the service offers, and summarize that for the user in plain language.

### Enable a service

This is a multi-step operation — do it all at once, don't make the user drive each step.

**Services are typically exported from the root workspace.** When the user says "enable X", assume the service lives at `root` unless you have reason to believe otherwise. Use the root workspace path (e.g., `root`, `root:<org>`) when creating the binding.

1. Find the service — check the root workspace first:

```bash
kubectl kcp ws root
kubectl get apiexport <service-name> -o yaml
kubectl kcp ws -
```

Note the service name. The workspace path for the binding is where you found it (usually `root` or `root:<org-name>`).

2. Check what permissions it needs:

```bash
kubectl get apiexport <service-name> -o jsonpath='{.spec.permissionClaims}'
```

(Run this from the workspace where the APIExport lives, i.e., root.)

3. Create the binding **in the user's workspace** with the root path and only the permissions the service requests:

```bash
kubectl apply -f - <<'EOF'
apiVersion: apis.kcp.io/v1alpha1
kind: APIBinding
metadata:
  name: <service-name>
spec:
  reference:
    export:
      name: <service-name>
      path: root
  permissionClaims:
    - all: true
      resources: <whatever-the-service-needs>
      state: Accepted
EOF
```

4. Confirm it worked:

```bash
kubectl get apibindings
```

Tell the user: "Done, I've enabled <service name>. You can now create <resource types it provides>."

If the kcp plugin is installed, you can simplify step 3:

```bash
kubectl kcp bind apiexport root:<service-name>
```

### Workspace types

There are two workspace types:

- kdp-organization: for organizations
- kdp-project: for individual projects inside of an organization

### Create a resource

1. Check the schema — always do this, don't guess:

```bash
kubectl explain <kind> --api-version=<group/version>
kubectl explain <kind>.spec --api-version=<group/version>
```

2. Build and apply a manifest from what the user asked for + what the schema requires:

```bash
kubectl apply -f - <<'EOF'
apiVersion: <group/version>
kind: <Kind>
metadata:
  name: <user-chosen-name>
spec:
  ...
EOF
```

If you create a resource that corresponds to a service/apiresourceschema and is written with crossplane please make sure to set the `spec.writeConnectionSecretToRef.name` otherwise this will cause issues.

3. Confirm:

```bash
kubectl get <kind> <name>
```

Tell the user what was created and what to expect next (e.g., "it's provisioning, status will update once it's ready").

### Check status

```bash
kubectl get <kind> <name> -o yaml
```

Read the `.status` section. Translate conditions into plain language — "it's ready", "still provisioning", "there's an error: <message>". Don't dump raw YAML at the user unless they ask for it.

### Get credentials

Resources often have related Secrets or ConfigMaps with connection info. Find them:

```bash
kubectl get <kind> <name> -o jsonpath='{.metadata.annotations}'
```

Look for `related-resources.kdp.k8c.io/*` annotations. These point to Secrets. List the keys:

```bash
kubectl get secret <related-name> -o jsonpath='{range .data}{@.key}{"\n"}{end}'
```

Tell the user what credentials are available (e.g., "there's a secret with keys: host, port, username, password"). If they need a specific value:

```bash
kubectl get secret <related-name> -o jsonpath='{.data.<key-name>}'
```

**Don't dump all secret values.** Only show what the user specifically asks for, and mention that it's base64-encoded.

### Manage resources

```bash
kubectl get <kind>                     # list all
kubectl describe <kind> <name>         # detailed status
kubectl apply -f updated.yaml          # update
kubectl delete <kind> <name>           # remove
```

### Navigate projects (needs kcp plugin)

```bash
kubectl kcp ws .              # current location
kubectl kcp ws <path>         # switch
kubectl kcp ws tree           # list sub-projects
kubectl kcp ws create <name>  # new project
```

Present these as "projects" to the user, not "workspaces".

## Rules

**Never use internal terms in your responses.** The mapping table above is exhaustive — if you catch yourself about to say APIBinding, APIExport, workspace, CRD, or anything kcp-specific, rephrase.

**Combine steps into single actions.** When the user says "enable the database service", do the full flow (find it, check permissions, create binding, confirm) and report back once. Don't ask the user to do intermediate steps.

**Always check the schema before creating resources.** `kubectl explain` is mandatory. Don't guess field names.

**Translate status into plain language.** Read `.status.conditions` and tell the user what's happening. "Your database is ready" or "Still setting up, give it a minute" — not "Phase: Bound, Ready: True".

**spec goes in, status comes back.** The user sets what they want in `spec`. The platform fills `status` with progress and results.

**If something fails**, troubleshoot internally:
- Is the service enabled? (`kubectl get apibindings`)
- Are permissions accepted? (check `.spec.permissionClaims` in the binding)
- Is the API group registered? (`kubectl api-resources`)

Then explain in plain language: "that service isn't enabled yet — want me to enable it?" Not "the APIBinding is missing".
