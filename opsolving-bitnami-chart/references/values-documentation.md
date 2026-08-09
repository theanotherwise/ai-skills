# Document Bitnami-style values

## Contents

- Purpose and precedence
- Core syntax
- Choose the documentation boundary
- Compact structured maps
- Configuration namespaces
- Component namespaces
- Recursive placement
- Opaque maps and lists
- Notes, examples, and references
- Patterns to avoid
- Decision procedure
- Review checklist

## Purpose and precedence

Make `values.yaml` read like a maintained Bitnami chart. Keep each parameter description close enough to its value that a reader can connect them without searching through a large comment wall.

Use this precedence:

1. Preserve a consistent documentation layout already used by the target chart.
2. Follow the closest current Bitnami chart with the same semantic structure.
3. Apply the rules and decision procedure below.

Do not mechanically normalize a chart only because another Bitnami chart made a different local choice. Bitnami charts contain historical variation; copy the structural intent, not every inconsistency.

## Core syntax

Use `##` comments for readme-generator metadata and documentation:

```yaml
## @section Traffic exposure parameters

## @param serviceType Kubernetes Service type
##
serviceType: ClusterIP
```

Use the complete path from the values root. Inside `master`, write `master.count`, not `count`. Inside `master.service`, write `master.service.type`, not `service.type`.

Indent a documentation block to the same level as the key it documents. Close a parameter block with `##` before the documented key unless an established local pattern deliberately omits it.

Document every public leaf value exactly once. A structural parent map does not need its own `@param` when its public children are documented individually. Use `@skip` only when a readme-generator path must intentionally remain undocumented.

## Choose the documentation boundary

Do not decide placement from YAML depth alone. Classify the current map by its role:

- A compact structured map is one cohesive value with a small fixed schema, such as a probe, security context, image definition, update strategy, or container ports.
- A configuration namespace is a large feature area containing independently useful options, nested structures, examples, or references, such as `service`, `persistence`, `networkPolicy`, `ingress`, or `serviceAccount`.
- A component namespace represents an independently configured workload or role, such as `master`, `replica`, `sentinel`, or `redis` in `redis-cluster`.
- An opaque value is a user-supplied map or list whose arbitrary contents the chart does not define, such as `podLabels`, `annotations`, `affinity`, `extraVolumes`, or `extraDeploy`.

Apply the classification recursively at every map. Root-level placement does not automatically require grouping, and nested placement does not automatically require per-key comments.

## Compact structured maps

Group all child `@param` lines immediately above a compact map when all of these conditions hold:

- the child schema is fixed and small enough to scan as one unit;
- most children need only a one-line description;
- the grouped block does not cross conceptual subgroups;
- the closest Bitnami reference uses the grouped form.

```yaml
## @param podSecurityContext.enabled Enable the Pod security context
## @param podSecurityContext.runAsNonRoot Require containers to run as non-root
## @param podSecurityContext.seccompProfile.type Set the seccomp profile type
##
podSecurityContext:
  enabled: true
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

Typical grouped maps include `image`, `containerPorts`, `startupProbe`, `livenessProbe`, `readinessProbe`, `podSecurityContext`, `containerSecurityContext`, `updateStrategy`, and small retention-policy or PDB objects.

Do not treat that list as unconditional. If a particular map has many independent options or extensive per-field notes, use the namespace layout instead.

## Configuration namespaces

Put each immediate child's documentation inside a configuration namespace, directly before that child:

```yaml
## Service parameters
##
service:
  ## @param service.type Kubernetes Service type
  ##
  type: ClusterIP
  ## @param service.ports.http Service HTTP port
  ## @param service.ports.https Service HTTPS port
  ##
  ports:
    http: 80
    https: 443
  ## @param service.annotations Additional Service annotations
  ##
  annotations: {}
```

Use this layout when grouping every descendant above `service:` would create a long wall detached from the values. Keep the namespace heading descriptive; do not add a redundant `@param service` when its children define the public API.

Large `persistence`, `networkPolicy`, `ingress`, `autoscaling`, `serviceAccount`, metrics, and external-access maps normally use this pattern.

## Component namespaces

Never put all `master.*`, `replica.*`, `sentinel.*`, or `redis.*` parameter lines above the component key. Start the component after its section heading, then document each immediate child inside it:

```yaml
## @section Redis master configuration parameters
##
master:
  ## @param master.count Number of master instances
  ##
  count: 1
  ## @param master.command Override the default command
  ##
  command: []
```

A component is an organizational namespace, not one compact value. Keeping documentation inside it makes role-specific options readable and prevents the component heading from accumulating hundreds of descendant paths.

## Recursive placement

Reclassify every nested map instead of inheriting the parent layout. A large component may contain a compact child:

```yaml
master:
  ## @param master.startupProbe.enabled Enable the startup probe
  ## @param master.startupProbe.periodSeconds Set the probe interval
  ## @param master.startupProbe.failureThreshold Set the failure threshold
  ##
  startupProbe:
    enabled: false
    periodSeconds: 10
    failureThreshold: 6
```

The same component may also contain a large namespace whose keys need individual comments:

```yaml
master:
  ## Persistence parameters
  ##
  persistence:
    ## @param master.persistence.enabled Enable persistent storage
    ##
    enabled: true
    ## @param master.persistence.storageClass Persistent Volume storage class
    ##
    storageClass: ""
    ## @param master.persistence.size Persistent Volume size
    ##
    size: 8Gi
```

Continue this rule at deeper levels. For example, a large `service` namespace may contain a compact `ports` map whose `service.ports.*` descriptions are grouped immediately above `ports:`.

## Opaque maps and lists

Document an arbitrary user-supplied object as one value. Do not invent child paths for keys the chart does not own:

```yaml
## @param podLabels Additional labels for application Pods
##
podLabels: {}

## @param extraVolumes Additional volumes for the application Pod
##
extraVolumes: []
```

Use type annotations such as `[object]`, `[array]`, or `[string]` when they clarify a value that the readme generator could infer incorrectly. Do not annotate every obvious scalar mechanically.

## Notes, examples, and references

Keep a reference, warning, default explanation, or example with the parameter it explains. Place it between the `@param` line and the closing `##`, or immediately before the `@param` when that matches the surrounding Bitnami block.

For a grouped compact map, extended usage notes may remain next to the relevant YAML key when moving them into the grouped header would make the header hard to scan. Do not repeat the `@param` line in both places.

Use examples only when the value shape or interaction is not obvious. Prefer a small commented YAML example over a long prose explanation. Explain precedence explicitly for pairs such as inline configuration versus `existingConfigmap` or generated credentials versus `existingSecret`.

## Patterns to avoid

Do not:

- place every descendant path of a component above `master:`, `replica:`, `redis:`, or another large namespace;
- group a long `service.*` or `persistence.*` API above the parent merely because the parent is at the root;
- put a one-line scalar description far above unrelated values;
- document a structural parent and all of its children redundantly;
- invent documentation for arbitrary keys inside user-provided maps;
- mix grouped and per-key `@param` placement randomly within one compact object;
- copy inconsistent legacy formatting when a clearer nearby Bitnami pattern exists;
- restate generic Kubernetes documentation without explaining a chart-specific decision.

Bad:

```yaml
## @param master.count Number of masters
## @param master.command Override the command
## @param master.service.type Service type
## @param master.persistence.enabled Enable persistence
## ...many unrelated master descendants...
##
master:
  count: 1
  command: []
  service: {}
  persistence: {}
```

## Decision procedure

For every public value:

1. Identify the nearest semantic parent.
2. If the parent is a component or large feature namespace, put the child's documentation inside the parent.
3. If the value is a compact fixed-schema map, group its concise child descriptions immediately above the map.
4. If the value is an opaque user-supplied map or list, document only the value itself.
5. If a nested map is encountered, repeat the procedure for that map.
6. If grouping produces a long or mixed-purpose comment wall, switch that map to per-key documentation.
7. Compare the result with the closest Bitnami reference: use `nginx` for root single-component values, `redis` for `master` and `replica`, and `redis-cluster` for the `redis` component namespace.

## Review checklist

- Every documented path is complete and matches an actual value.
- Each public leaf or opaque value has exactly one owning documentation block.
- Component and large feature options are documented inside their namespace.
- Compact maps use grouped documentation only while the block remains cohesive and scannable.
- Nested maps were classified recursively rather than formatted only by indentation depth.
- Comments have the same indentation as their owning key.
- Examples, references, warnings, and precedence notes remain next to the relevant value.
- No long comment wall is detached from the values it describes.
- The final layout resembles the closest Bitnami reference without copying unrelated options.
