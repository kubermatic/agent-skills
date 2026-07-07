# Writing the resourceGraph (kro ResourceGraphDefinition)

The `spec.resourceGraph` of a `BlueprintDefinition` is a kro `ResourceGraphDefinition` (RGD)
stored verbatim. This is the actual composition logic. Read this when you're writing anything
beyond a trivial graph.

## Contents

- [Top-level structure](#top-level-structure)
- [The schema block](#the-schema-block) — defining the synthesized kind
- [Schema type markers](#schema-type-markers)
- [The resources list](#the-resources-list)
- [CEL references and dependency edges](#cel-references-and-dependency-edges)
- [externalRef — reading resources you don't create](#externalref)
- [readyWhen — gating on a dependency](#readywhen)
- [includeWhen — conditional inclusion](#includewhen)
- [forEach — collections](#foreach)
- [Routing connection secrets safely](#routing-connection-secrets-safely)
- [Naming: the consumer API group](#naming-the-consumer-api-group)

## Top-level structure

```yaml
apiVersion: kro.run/v1alpha1
kind: ResourceGraphDefinition
metadata:
  name: postgrespair            # lowercase; conventionally the plural-ish kind name
spec:
  schema:                       # the synthesized kind users will create
    apiVersion: v1alpha1
    kind: PostgresPair
    spec: { ... }
    status: { ... }
  resources:                    # the child CRs this Blueprint composes
    - id: dbA
      template: { ... }
    - id: dbB
      template: { ... }
```

## The schema block

`spec.schema` defines the new kind that gets published. `kind` + `apiVersion` name it; `spec`
declares the fields a user fills in when creating an instance; `status` declares fields you
surface back (populated via CEL from the children).

```yaml
schema:
  apiVersion: v1alpha1
  kind: PostgresPair
  spec:
    name: string | required=true
    size: string | default="1Gi"
  status:
    # surface a child's live value back on the instance
    primaryHost: ${dbA.status.host}
```

The user's instance spec is then addressable in `resources` as `${schema.spec.<field>}`.

## Schema type markers

kro declares field types with a string marker syntax (`type | marker=value ...`), not full
OpenAPI. Common forms:

| Marker | Meaning |
|---|---|
| `string`, `integer`, `boolean`, `number` | scalar types |
| `required=true` | field must be set |
| `default="1Gi"` / `default=3` | default when omitted |
| `description="..."` | human description |
| `[]string` | array of scalars |
| nested map | object — declare sub-fields as a nested mapping |

```yaml
spec:
  name: string | required=true description="instance name prefix"
  size: string | default="1Gi"
  replicas: integer | default=2
  tags: "[]string"
  tuning:                       # nested object
    sharedBuffers: string | default="128MB"
```

## The resources list

Each entry is a child resource the Blueprint applies for every instance:

- `id` — a unique identifier used to reference this node from CEL (`${dbA.status...}`) and
  shown in `status.waitingOn`. Keep it short and meaningful.
- `template` — a complete child CR (`apiVersion`, `kind`, `metadata`, `spec`). Every
  `(apiVersion, kind)` here must be **bound in the authoring workspace** or validation fails.

```yaml
resources:
  - id: dbA
    template:
      apiVersion: dbms.example.corp/v1alpha1
      kind: PostgresInstance
      metadata:
        name: ${schema.spec.name}-a       # derive names from the instance spec
      spec:
        storageSize: ${schema.spec.size}
```

## CEL references and dependency edges

Wire values with `${...}` CEL expressions:

- `${schema.spec.<field>}` — read the instance's spec.
- `${<id>.status.<field>}` / `${<id>.metadata.<field>}` — read another child's **live**
  value. kro infers a dependency edge from the reference, so `dbB` referencing
  `${dbA.status.host}` is applied only after `dbA` is observed.
- Expressions can use CEL functions and string concatenation, e.g.
  `${schema.spec.name + "-replica"}`.

The graph must be **acyclic** — a reference cycle between resources fails validation.

A node that references a not-yet-available value (a dependency's `status` that hasn't been
populated) is *data-pending*: it's skipped and requeued, and resolves on a later pass once the
dependency is observed. This is why a Blueprint instance can take several reconcile passes to
settle.

## externalRef

`externalRef` reads a resource the Blueprint does **not** create — typically to route a
composed service's connection info (a Secret/ConfigMap surfaced via
`related-resources.kdp.k8c.io/*` annotations) into a downstream child. External nodes are
read-only: never applied, owned, or pruned.

Two forms:

```yaml
resources:
  - id: dbSecret
    externalRef:                    # scalar: one object by name (defaults to instance namespace)
      apiVersion: v1
      kind: Secret
      metadata:
        name: ${dbA.status.connectionSecretName}

  - id: allConfigs
    externalRef:                    # collection: by label selector
      apiVersion: v1
      kind: ConfigMap
      selector:
        matchLabels:
          app: ${schema.spec.name}
```

A downstream resource then references the external node like any other: `${dbSecret.data.host}`
(prefer routing the *name* — see below).

## readyWhen

`readyWhen` gates a node: dependents aren't applied until the condition holds. Use it for slow
dependencies (e.g. a ~40s Crossplane provisioning wait) so a consumer doesn't get applied
against a not-yet-ready database.

```yaml
resources:
  - id: dbA
    readyWhen:
      - ${dbA.status.conditions.exists(c, c.type == "Ready" && c.status == "True")}
    template:
      apiVersion: dbms.example.corp/v1alpha1
      kind: PostgresInstance
      # ...
```

An unsatisfied `readyWhen` keeps the whole instance unsettled (`status.ready` stays `False`,
the node id appears in `status.waitingOn`).

## includeWhen

`includeWhen` conditionally includes a node based on the instance spec. An excluded node is
pruned (deleted) if it was previously created.

```yaml
resources:
  - id: replica
    includeWhen:
      - ${schema.spec.replicas > 1}
    template:
      # ...
```

Note: if the condition can't be evaluated yet (data-pending), the node is **not** pruned —
pruning a prior child is deferred until the condition is definitively false, so a transient
unevaluable state never deletes a healthy child.

## forEach

`forEach` expands a node into a collection from a list in the instance spec:

```yaml
resources:
  - id: db
    forEach: ${schema.spec.databases}      # a []object / []string in the schema
    template:
      apiVersion: dbms.example.corp/v1alpha1
      kind: PostgresInstance
      metadata:
        name: ${schema.spec.name}-${forEach.value.name}
      spec:
        storageSize: ${forEach.value.size}
```

## Routing connection secrets safely

When a downstream child needs another service's credentials, prefer **reference-routing**:
pass the Secret's **name** (e.g. `${dbA.status.connectionSecretName}`) so the consumer mounts
the existing Secret. Avoid **data-copy** — reading Secret `data` into CEL and building a new
object — because it duplicates secret material into a new object and is gated behind an
explicit decision. Non-secret reads (ConfigMap values, CR status) are fine to route through
CEL. A controller-created merged Secret must carry an owner reference so it's garbage-collected
with the instance.

## Naming: the consumer API group

The published synthesized kind's API group is derived as
`<kind-lowercased>.blueprints.kdp.k8c.io`. So `kind: PostgresPair` →
`postgrespair.blueprints.kdp.k8c.io/v1alpha1`, and the generated `APIExport` and catalog
`Service` are both named `postgrespair.blueprints.kdp.k8c.io`.

The `APIResourceSchema` name is **content-addressed** — `<version-slug>-<hash>.<plural>.<group>`
(e.g. `v0-1-0.postgrespairs.postgrespair.blueprints.kdp.k8c.io`) — because schemas are
immutable once created. That's why changing a published Blueprint requires a new `version`:
a new version yields a new schema name.
