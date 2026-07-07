---
name: kdp-blueprints
description: Compose multiple Kubermatic Developer Platform (KDP) services into a single reusable Blueprint, then validate and publish it to the service catalog. Use this whenever a developer wants to combine, bundle, or wire together several KDP services (e.g. a database + a repo + an app) into one provisionable kind, author or edit a BlueprintDefinition, write a kro ResourceGraphDefinition for KDP, debug why a Blueprint won't validate or publish, or troubleshoot a Blueprint instance that isn't becoming ready — even if they don't say the word "Blueprint". This is for Blueprint *authors*; for just enabling/creating individual services, use the kdp or kdp-kubectl skills instead.
allowed-tools:
  - "Bash(kubectl *)"
---

# Composing KDP Blueprints

You're helping a developer **compose KDP services into a Blueprint** — a reusable bundle
that wires several existing services together into one new provisionable kind. For example,
a `PostgresPair` Blueprint that creates two `PostgresInstance`s from a single short spec, or
a `WebappStack` that provisions a database + a git repo + an app together.

This skill is for **Blueprint authors**, so it uses the real platform terminology
(BlueprintDefinition, ResourceGraphDefinition, APIBinding, kro, CEL). That's a deliberate
contrast with the `kdp` skill, which hides internals from end users. If the user only wants
to *enable* a service or *create* a single resource, that's the `kdp` / `kdp-kubectl` skill —
not this one.

Everything here is plain `kubectl` against a KDP kubeconfig.

## The mental model: two loops

A Blueprint has two phases. Keep them straight — most confusion comes from mixing them up.

1. **Authoring loop.** You write a `BlueprintDefinition` containing a kro
   `ResourceGraphDefinition` (the composition). The controller validates it
   (`Draft → Valid | Invalid`) and, when you flip `published: true`, emits three artifacts:
   an `APIResourceSchema`, an `APIExport`, and a catalog `Service`. The synthesized kind
   (e.g. `PostgresPair`) now exists and shows up in the catalog.

2. **Instance loop.** Once published, anyone who binds the Blueprint can create *instances*
   of the synthesized kind. For each instance, the controller renders your RGD and
   server-side-applies the child service CRs into the same workspace, wiring their live
   values together. The instance's `status.ready` reflects the children's readiness.

As the author you mostly drive loop 1; you trigger loop 2 once at the end to smoke-test.

## Prerequisite: bind every service you want to compose

A Blueprint can only compose services that are **bound in the authoring workspace**. The
validator walks every `(apiVersion, kind)` in your RGD and checks that the API is available
here — a kind that doesn't resolve almost always means that service isn't bound yet.

Confirm the building-block kinds are available before you write anything:

```bash
kubectl api-resources | grep <service-group>     # e.g. dbms.example.corp
```

You also need the authoring API itself bound — `blueprints.kdp.k8c.io` (it provides the
`BlueprintDefinition` kind). Check:

```bash
kubectl api-resources | grep blueprints.kdp.k8c.io
```

If a building-block service isn't bound, enable it first using the **`kdp-kubectl` skill's
"enable a service" flow** (create an `APIBinding` / `kubectl kcp bind apiexport`). Don't
duplicate that here — bind it, confirm with `api-resources`, then come back.

## Anatomy of a BlueprintDefinition

`BlueprintDefinition` (`blueprints.kdp.k8c.io/v1alpha1`) is **namespaced**. Skeleton:

```yaml
apiVersion: blueprints.kdp.k8c.io/v1alpha1
kind: BlueprintDefinition
metadata:
  name: postgres-pair
  namespace: default
spec:
  version: v0.1.0          # semver; IMMUTABLE once published — bump it for changes
  displayName: Postgres Pair
  description: Two Postgres databases sharing a common storage size
  category: Databases       # catalog grouping
  # logoURL: data:image/png;base64,...   # optional, shown in the dashboard catalog
  published: false          # start false; flip to true once it validates
  # deprecated: false       # set true to block NEW instances (existing ones keep running)
  catalogMetadata:          # mirrored verbatim onto the generated catalog Service
    title: Postgres Pair
    description: Two Postgres databases sharing a common storage size
    category: Databases
  resourceGraph:            # the kro ResourceGraphDefinition — the actual composition
    apiVersion: kro.run/v1alpha1
    kind: ResourceGraphDefinition
    # ... see "Writing the resourceGraph" below
```

Field notes that bite people:

- **`version` is immutable once published.** To change a published Blueprint, bump the
  version — don't mutate the existing one. The version also feeds the content-addressed
  schema name, so each version is a distinct schema.
- **`published: true` is one-way for artifacts.** Flipping back to `false` stops instance
  reconciliation but does **not** delete the already-published `APIResourceSchema` /
  `APIExport` / `Service`. Cleaning those up is manual.
- **`deprecated: true`** blocks new instances but leaves existing ones running.
- `spec` goes in, `status` comes back — never hand-edit `status` or the generated artifacts.

## Writing the resourceGraph (the kro RGD)

The `resourceGraph` is a kro `ResourceGraphDefinition` stored verbatim. It has two parts:

- **`spec.schema`** — defines the synthesized kind: its `kind`, `apiVersion`, and the
  `spec`/`status` fields a user fills in. The consumer API group is derived automatically as
  `<kind-lowercased>.blueprints.kdp.k8c.io`.
- **`spec.resources`** — the list of child CRs to create, each with an `id` and a `template`
  (a full service CR). You wire fields together with CEL: `${schema.spec.size}` pulls from
  the instance's spec, `${dbA.status.host}` pulls another child's live value (kro infers the
  dependency edge from the reference).

Minimal shape (full worked example in `examples/postgres-pair.yaml`):

```yaml
resourceGraph:
  apiVersion: kro.run/v1alpha1
  kind: ResourceGraphDefinition
  metadata:
    name: postgrespair
  spec:
    schema:
      apiVersion: v1alpha1
      kind: PostgresPair
      spec:
        name: string | required=true
        size: string | default="1Gi"
    resources:
      - id: dbA
        template:
          apiVersion: dbms.example.corp/v1alpha1
          kind: PostgresInstance
          metadata:
            name: ${schema.spec.name}-a
          spec:
            storageSize: ${schema.spec.size}
      - id: dbB
        template:
          apiVersion: dbms.example.corp/v1alpha1
          kind: PostgresInstance
          metadata:
            name: ${schema.spec.name}-b
          spec:
            storageSize: ${schema.spec.size}
```

kro has a few features that make compositions powerful — `externalRef`, `readyWhen`,
`includeWhen`, `forEach`, and cross-resource references. **Read
`reference/rgd-authoring.md` before writing anything beyond a trivial two-child graph** —
it covers the schema type markers, CEL reference forms, and each feature with examples.

## The author workflow

Drive this end to end; don't make the user run each step.

1. **Confirm building blocks are bound** (see prerequisite above). If not, bind them first.

2. **Author the RGD and create the BlueprintDefinition with `published: false`.** Starting
   unpublished lets you fix validation errors without emitting half-baked catalog artifacts.

   ```bash
   kubectl apply -f - <<'EOF'
   apiVersion: blueprints.kdp.k8c.io/v1alpha1
   kind: BlueprintDefinition
   metadata:
     name: postgres-pair
     namespace: default
   spec:
     version: v0.1.0
     published: false
     # ...resourceGraph...
   EOF
   ```

3. **Validate — read the structured errors, don't guess.**

   ```bash
   kubectl get blueprintdefinition postgres-pair -o jsonpath='{.status.phase}{"\n"}'
   kubectl get blueprintdefinition postgres-pair -o jsonpath='{.status.validationErrors}' | jq .
   ```

   `status.validationErrors` carries a `path` (e.g. `resources[1].template.spec.storageSize`)
   and a `message` for each problem — fix against those exact paths, re-apply, repeat until
   `status.phase` is `Valid` and the `Valid` condition is `True`.

4. **Publish.** Set `published: true` and re-apply. The controller needs a few seconds to
   emit the artifacts — phase transitions from `Valid` to `Published` asynchronously. Poll
   until it settles:

   ```bash
   kubectl get blueprintdefinition postgres-pair -o jsonpath='{.status.phase}{"\n"}'  # Published (wait ~10-15s)
   kubectl get blueprintdefinition postgres-pair -o jsonpath='{.status.apiExportRef}' # populated
   kubectl get apiexport <kind-lower>.blueprints.kdp.k8c.io
   kubectl get services.core.kdp.k8c.io <kind-lower>.blueprints.kdp.k8c.io
   ```

5. **Smoke-test as a consumer.** Bind the new Blueprint kind (its `APIExport` lives in your
   workspace — use the `kdp-kubectl` bind flow) and create one instance, then watch it
   render:

   ```bash
   kubectl apply -f - <<'EOF'
   apiVersion: postgrespair.blueprints.kdp.k8c.io/v1alpha1
   kind: PostgresPair
   metadata:
     name: pair-svc
     namespace: default
   spec:
     name: pair-svc
     size: 1Gi
   EOF

   kubectl get postgrespair pair-svc -o yaml   # watch status.ready / status.waitingOn / childRefs
   ```

   A settled instance shows `status.ready: "True"` and lists each child under
   `status.childRefs` with its own conditions. `status.waitingOn` lists blocking node IDs
   while it's still converging.

6. **Iterate / lifecycle.** To change a published Blueprint, **bump `version`** and publish
   the new one (versions are immutable). Set `deprecated: true` to stop new instances.

## Inspect & troubleshoot

```bash
kubectl get blueprintdefinitions                       # Version / Phase / Published columns
kubectl get blueprintdefinition <name> -o yaml         # full status
kubectl get blueprintdefinition <name> -o jsonpath='{.status.validationErrors}' | jq .
```

| Symptom | Likely cause | What to check |
|---|---|---|
| Phase `Invalid` | RGD failed validation | `status.validationErrors` — exact field paths |
| "kind not found" / unresolved type | composed service not bound here | `kubectl api-resources \| grep <group>`; bind it |
| Stuck `Validating` | controller still working / transient | re-check phase; look at conditions |
| Published but not in catalog | `Service` artifact missing | `kubectl get services.core.kdp.k8c.io` |
| Instance not `ready` | a child isn't ready / gated | instance `status.waitingOn`, then each child in `status.childRefs` |
| Can't create an instance | Blueprint `deprecated`, or not bound | check `spec.deprecated`; confirm the synthesized kind is bound |

To find an instance's children and their health:

```bash
kubectl get <synthesized-kind> <name> -o jsonpath='{.status.childRefs}' | jq .
```

## Rules

- **Bind every composed service first.** Validation resolves each child kind against the
  workspace; an unbound service is the #1 cause of `Invalid`.
- **Read `status.validationErrors` before guessing.** They give exact field paths — far
  faster than re-reading the RGD by eye.
- **`version` is immutable once published — bump, don't mutate.**
- **Start `published: false`, flip to `true` only once `Valid`.** Unpublishing doesn't
  remove already-emitted artifacts.
- **Prefer reference-routing of Secrets over copying data.** When a child needs another's
  credentials, pass the Secret's *name* downstream (via `externalRef` / a field ref) rather
  than reading Secret `data` into CEL and rebuilding it. See `reference/rgd-authoring.md`.
- **Never hand-edit the generated artifacts or an instance's child CRs.** The controller
  owns them; `spec` goes in, `status` comes back.
- **This skill is for authoring.** For enabling a service or creating a one-off resource,
  use the `kdp` / `kdp-kubectl` skills.
