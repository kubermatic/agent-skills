---
name: kdp-kubectl
description: Interact with Kubermatic Developer Platform (KDP) using kubectl. Use this when the user wants to discover services, enable them, create or manage resources, or explore their platform environment. No MCP servers needed — just kubectl with a KDP kubeconfig.
allowed-tools:
  - "Bash(kubectl *)"
---

# KDP with kubectl

You're helping a developer work with their Kubermatic Developer Platform (KDP) environment using `kubectl`. Everything in KDP is exposed as Kubernetes APIs, so standard kubectl works for all operations.

## Check connectivity

```bash
kubectl api-resources
```

If this returns results, you're good. Custom API groups (like `databases.example.corp`, `kkp.kdp.io`) are services registered on the platform.

## Browse the service catalog

```bash
# See all services you can enable
kubectl get apiexports

# Details on a specific service
kubectl get apiexport <service-name> -o yaml
```

## Enable a service

With the kcp kubectl plugin (if installed):

```bash
kubectl kcp bind apiexport root:my-org:<service-name>
```

Without the plugin — create an APIBinding:

```yaml
apiVersion: apis.kcp.io/v1alpha1
kind: APIBinding
metadata:
  name: <service-name>
spec:
  reference:
    export:
      name: <service-name>
      path: <workspace-path>
  permissionClaims:
    - all: true
      resources: secrets
      state: Accepted
    - all: true
      resources: configmaps
      state: Accepted
```

```bash
kubectl apply -f apibinding.yaml
```

Most services need access to Secrets and ConfigMaps to deliver credentials. Set `state: Accepted` for what the service requests, otherwise it won't work properly.

## Check enabled services

```bash
kubectl get apibindings
kubectl get apibinding <name> -o yaml   # check status.conditions
```

## Disable a service

```bash
kubectl delete apibinding <name>
```

## Look up a resource schema

Before creating anything, check what fields it expects:

```bash
kubectl explain <kind> --api-version=<group/version>
kubectl explain <kind>.spec --api-version=<group/version>
```

Example:

```bash
kubectl explain database --api-version=databases.example.corp/v1
kubectl explain database.spec --api-version=databases.example.corp/v1
```

## Create a resource

1. Look up the schema with `kubectl explain`
2. Write a manifest
3. Apply it

```bash
kubectl apply -f - <<'EOF'
apiVersion: databases.example.corp/v1
kind: Database
metadata:
  name: my-database
spec:
  engine: postgresql
  version: "16"
  storage:
    size: 10Gi
EOF
```

## Manage resources

```bash
kubectl get <kind>                          # list all
kubectl get <kind> <name> -o yaml           # full details
kubectl describe <kind> <name>              # status + events
kubectl apply -f updated-resource.yaml      # update
kubectl delete <kind> <name>                # remove
```

## Get credentials and related resources

Services often create Secrets or ConfigMaps alongside the main resource. Check annotations:

```bash
kubectl get <kind> <name> -o jsonpath='{.metadata.annotations}'
```

Look for annotations starting with `related-resources.kdp.k8c.io/` — these contain JSON with the name, namespace, apiVersion, and kind of related resources (usually Secrets with connection strings, passwords, etc). Fetch those with:

```bash
kubectl get secret <related-name> -o yaml
```

## Navigate workspaces (needs kcp plugin)

```bash
kubectl kcp ws .              # where am I?
kubectl kcp ws <path>         # switch workspace
kubectl kcp ws tree           # list children
kubectl kcp ws create <name>  # new project
```

## Rules

**Always check the schema first.** Run `kubectl explain` before creating resources. Don't guess at field names or structure — services define their own APIs.

**Custom API groups are normal.** `databases.example.corp/v1`, `kkp.kdp.io/v1alpha1`, etc. Always use `--api-version` with `kubectl explain` and the right `apiVersion` in manifests.

**spec goes in, status comes back.** Users set `spec`, the service fills `status` with provisioning info. Don't touch `status`.

**Permission claims matter.** If a service isn't working after binding, check that you accepted its permission claims (`spec.permissionClaims` in the APIBinding).

**Platform objects use `core.kdp.k8c.io/v1alpha1`.** Services, org settings, and platform config live under this API group.

**If something fails**, check: is the service enabled? (`kubectl get apibindings`) Are permission claims accepted? Is the API group right? (`kubectl api-resources`)
