---
name: bitnami-chart
description: Create and refactor Helm charts in Bitnami's structural style while using the Opsolving common library instead of Bitnami's repository or OCI registry. Use when designing Chart.yaml, values.yaml, templates, _helpers.tpl, single-component or multi-component charts, or when reviewing an existing chart for conformance with these conventions.
---

# Build Bitnami-style Helm charts with Opsolving common

## Goal

Create charts that look and behave like well-maintained Bitnami charts while sourcing the `common` library from `https://opsolving.github.io/charts/`.

Follow Bitnami's component modeling, `values.yaml` organization, `templates/` layout, variable locality, and helper usage. Do not blindly copy names, defaults, images, copyright notices, or features that do not fit the target application.

## Source precedence and compatibility

Use this source order:

1. the user's requirements and the target repository's established conventions;
2. the published `opsolving/common` API;
3. structural patterns from Bitnami charts, especially `redis`, `redis-cluster`, `nginx`, `postgresql`, `mariadb`, and `flink`;
4. the official Kubernetes and Helm APIs.

Treat `opsolving/common` as the compatibility boundary. Do not assume that every helper found on the current Bitnami `main` branch is available. Opsolving currently publishes `common` 2.31.4, so confirm that each helper exists in that version before using it.

Do not use helpers introduced only by a newer Bitnami `common`. For example, do not assume that `common.labels.value` is available; it is absent from Opsolving `common` 2.31.4.

## Required workflow

1. Read the governing `AGENTS.md`, existing `Chart.yaml`, `values.yaml`, templates, schemas, and chart tests.
2. Identify the application's actual processes, roles, images, ports, volumes, secrets, and Kubernetes resources.
3. Decide whether the chart has one workload component or several independently configurable components.
4. Design the public `values.yaml` API before writing templates.
5. Add the `opsolving/common` dependency and call its helpers instead of copying them locally.
6. Create one readable template per resource or tightly related resource group.
7. Add local helpers only after a concrete chart-specific need appears.
8. Verify every `.Values` path, selector, name, condition, templated value, and `common` helper signature.
9. Run the narrowest available checks without deploying the chart or starting services.

## Choose the component model

Classify the chart before creating files. Do not infer a separate component merely from the presence of a metrics sidecar, init container, or helper job.

### Single workload component

When the application has one main Deployment, StatefulSet, or DaemonSet and one set of pod settings, keep workload settings at the root of `values.yaml`.

Use a shape such as:

```yaml
image: {}
replicaCount: 1
updateStrategy: {}
podLabels: {}
podAnnotations: {}
podSecurityContext: {}
containerSecurityContext: {}
resourcesPreset: none
resources: {}
service: {}
persistence: {}
metrics: {}
```

Do not add a redundant `.Values.app`, `.Values.server`, `.Values.workload`, or `.Values.deployment` level when it distinguishes no role. Flat charts such as `nginx`, `apache`, `memcached`, and `rabbitmq` are useful references.

Keep `templates/` flat while all resources belong to the same component:

```text
templates/
|-- _helpers.tpl
|-- deployment.yaml
|-- service.yaml
|-- serviceaccount.yaml
|-- configmap.yaml
|-- ingress.yaml
|-- networkpolicy.yaml
`-- servicemonitor.yaml
```

### Multiple components or roles

When roles are deployed, scaled, or configured independently, create one values block per role and a matching directory in `templates/`.

For example, model replicated Redis-like workloads as:

```yaml
architecture: replication

auth:
  enabled: true
  existingSecret: ""

master:
  replicaCount: 1
  podLabels: {}
  podAnnotations: {}
  resourcesPreset: none
  resources: {}
  service: {}
  persistence: {}

replica:
  replicaCount: 3
  podLabels: {}
  podAnnotations: {}
  resourcesPreset: none
  resources: {}
  service: {}
  persistence: {}

sentinel:
  enabled: false
  replicaCount: 3
  service: {}
```

Use a corresponding layout:

```text
templates/
|-- _helpers.tpl
|-- master/
|   |-- application.yaml
|   |-- service.yaml
|   |-- serviceaccount.yaml
|   |-- pdb.yaml
|   `-- pvc.yaml
|-- replicas/
|   |-- application.yaml
|   |-- service.yaml
|   |-- serviceaccount.yaml
|   `-- pdb.yaml
`-- sentinel/
    |-- statefulset.yaml
    |-- service.yaml
    `-- pdb.yaml
```

Put only genuinely role-specific settings inside component blocks: replica counts, update strategies, resources, scheduling, service accounts, probes, Services, persistence, extra environment variables, and volumes.

Keep system-wide settings at the root: `global`, a shared image, architecture, authentication, TLS, shared configuration, network policy, RBAC, a shared service account, metrics, volume permissions, and `extraDeploy`.

Do not duplicate the same setting at the root and in every component unless there is a documented override relationship. Keep a shared value at the root; move it into components only when their behavior must differ.

### Redis Cluster is not a master/replica values model

Distinguish logical roles from independently rendered workload types. `redis-cluster` renders one StatefulSet pod type and assigns master and replica roles inside the cluster. Its templates therefore stay flat, pod settings live under `redis`, and topology lives under `cluster`:

```yaml
redis:
  podLabels: {}
  resources: {}
  affinity: {}

cluster:
  nodes: 6
  replicas: 1
  externalAccess: {}
```

Do not create `master/` and `replica/` template directories when the chart does not render distinct workloads for those roles. For an ordinary one-component service without cluster topology, still prefer root-level workload values.

## Design values.yaml

### Section order

Use a Bitnami-like order:

1. global overrides;
2. common chart parameters;
3. images;
4. architecture, authentication, and application configuration;
5. single-workload parameters or component blocks;
6. Service, ingress, persistence, and network policy;
7. service account, RBAC, and security policies;
8. metrics and observability integrations;
9. helper init containers such as volume permissions or sysctl;
10. `extraDeploy` and remaining extensions when not already placed in common parameters.

Preserve the logical order even when the chart does not need every section.

### Document parameters

Document public values immediately before their definitions. Use the `readme-generator-for-helm` convention:

```yaml
## @section Service parameters

## @param service.type Kubernetes Service type
## @param service.ports.http Service HTTP port
## @param service.annotations Additional custom annotations for the Service
##
service:
  type: ClusterIP
  ports:
    http: 8080
  annotations: {}
```

Use the complete parameter path, such as `master.persistence.size`, rather than only `size`. Add examples when the accepted shape is not obvious. Do not restate Kubernetes documentation unless it helps chart users make a decision.

### Use familiar names and stable types

Prefer recognizable Bitnami names and Kubernetes-shaped values:

```yaml
global:
  imageRegistry: ""
  imagePullSecrets: []
  defaultStorageClass: ""
  compatibility:
    openshift:
      adaptSecurityContext: auto

nameOverride: ""
fullnameOverride: ""
namespaceOverride: ""
commonLabels: {}
commonAnnotations: {}
clusterDomain: cluster.local
extraDeploy: []

image:
  registry: docker.io
  repository: owner/application
  tag: "1.0.0"
  digest: ""
  pullPolicy: IfNotPresent
  pullSecrets: []

podSecurityContext:
  enabled: true
  fsGroup: 1001

containerSecurityContext:
  enabled: true
  runAsUser: 1001
  runAsGroup: 1001
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

resourcesPreset: none
resources: {}
startupProbe: {}
livenessProbe: {}
readinessProbe: {}
customStartupProbe: {}
customLivenessProbe: {}
customReadinessProbe: {}
extraEnvVars: []
extraEnvVarsCM: ""
extraEnvVarsSecret: ""
extraVolumes: []
extraVolumeMounts: []
sidecars: []
initContainers: []
```

Do not add this entire surface mechanically. Add only values supported by the templates and useful to the application.

Keep empty value types stable: maps as `{}`, lists as `[]`, strings as `""`, and booleans as `true` or `false`. Never change a parameter's type according to how it is used.

Do not use `global` as a miscellaneous settings bucket. A global value must actually override several images, subcharts, or components and have documented precedence.

Do not expose a value that no template reads. Do not read an undeclared value unless a deliberate, backward-compatible fallback requires it.

## Configure Chart.yaml and Opsolving common

Declare the dependency using this repository:

```yaml
apiVersion: v2
name: my-chart
description: A Helm chart for Kubernetes
type: application
version: 1.0.0
appVersion: "1.0.0"

dependencies:
  - name: common
    repository: https://opsolving.github.io/charts/
    version: 2.x.x
```

Use `2.x.x` by default, matching the published Opsolving example. Pin an exact published version only when the target repository's dependency policy explicitly requires deterministic pins; do not infer that requirement merely from the currently published version.

Do not use Bitnami's OCI location for `common`:

```yaml
repository: oci://registry-1.docker.io/bitnamicharts
```

Do not handwrite `Chart.lock` or copy a `common` archive from the Bitnami repository. If the target repository tracks a lockfile or vendored dependencies, generate them with Helm only after dependency fetching is authorized.

## Use Opsolving common directly

Call `common` at the rendering site. Commonly use `common.names.*`, `common.labels.standard`, `common.labels.matchLabels`, `common.images.*`, `common.tplvalues.*`, `common.capabilities.*`, `common.compatibility.renderSecurityContext`, `common.affinities.*`, `common.storage.class`, and `common.resources.preset`.

For secrets, use `common.secrets.name`, `common.secrets.key`, `common.secrets.passwords.manage`, and `common.secrets.lookup`. Use ingress, validation, warning, error, and utility helpers only after confirming their exact Opsolving 2.31.4 signatures.

Render names and namespaces without local copies:

```yaml
metadata:
  name: {{ include "common.names.fullname" . }}
  namespace: {{ include "common.names.namespace" . | quote }}
```

Merge global and component labels where the resource needs them:

```yaml
{{- $podLabels := include "common.tplvalues.merge" (dict "values" (list .Values.podLabels .Values.commonLabels) "context" .) }}
selector:
  matchLabels: {{- include "common.labels.matchLabels" (dict "customLabels" $podLabels "context" $) | nindent 4 }}
```

For a component, add the same fixed role label to resource metadata, workload selectors, pod template labels, and Service selectors:

```yaml
app.kubernetes.io/component: master
```

Render user values that may contain Helm syntax:

```yaml
annotations: {{- include "common.tplvalues.render" (dict "value" .Values.service.annotations "context" $) | nindent 4 }}
```

Do not replace `common.tplvalues.render` with `toYaml` when the public value is documented as templatable. Use plain `toYaml` for internal structures that intentionally do not support templating.

## Keep template variables local

Define a variable in the narrowest scope that covers its real uses. Do not create a top-of-file variable preamble merely to shorten later references.

Use `.Values.service.type` directly when it appears once. Define `$annotations` immediately before `metadata.annotations`. Define `$podLabels` immediately before the selector when the same merged labels are also needed by the pod template.

Preferred:

```yaml
metadata:
  name: {{ include "common.names.fullname" . }}
  {{- if or .Values.service.annotations .Values.commonAnnotations }}
  {{- $annotations := include "common.tplvalues.merge" (dict "values" (list .Values.service.annotations .Values.commonAnnotations) "context" .) }}
  annotations: {{- include "common.tplvalues.render" (dict "value" $annotations "context" $) | nindent 4 }}
  {{- end }}
```

Avoid:

```yaml
{{- $name := include "common.names.fullname" . }}
{{- $service := .Values.service }}
{{- $annotations := .Values.service.annotations }}
```

Do not create a variable for a value used once. Do not alias entire `.Values` blocks for convenience. Do not mutate `.Values` with `set` unless a documented compatibility or secret-state requirement makes it necessary.

Introduce `$root := .` immediately before a `range` or `with` when the changed dot context would hide the chart root. Hoist a variable only when an entire loop or several distant sections genuinely share it.

```yaml
{{- $root := . }}
{{- range $index := until (int .Values.replicaCount) }}
metadata:
  name: {{ include "common.names.fullname" $root }}-{{ $index }}
{{- end }}
```

## Keep _helpers.tpl chart-specific

Start with an empty `_helpers.tpl`, or omit it, until chart-specific logic is needed. Do not treat boilerplate helpers as mandatory scaffolding.

Add a helper only when at least one condition is true:

- it encodes a chart-domain rule such as a component name, secret key, architecture mode, or configuration choice;
- several templates use the same nontrivial transformation;
- it preserves a stable resource name referenced by many resources;
- it aggregates validations involving several values;
- it preserves Helm context that cannot be passed clearly inline.

Reasonable helpers include:

```gotemplate
{{- define "my-chart.master.fullname" -}}
{{- printf "%s-master" (include "common.names.fullname" .) | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "my-chart.secretName" -}}
{{- if .Values.auth.existingSecret -}}
{{- tpl .Values.auth.existingSecret $ -}}
{{- else -}}
{{- include "common.names.fullname" . -}}
{{- end -}}
{{- end -}}
```

Do not add local copies of name, fullname, chart, labels, selectors, namespace, tpl rendering, affinity, or API-version helpers because `common` already provides them.

Do not add a helper that merely renames a `common` helper without binding chart-specific values. A thin image helper is justified only when it removes a repeated chart-specific argument dictionary or is shared by several resources.

Prefix helper names with the chart and component, for example `my-chart.replica.fullname`. Put a short contract comment above each helper; do not comment on obvious syntax.

Do not place ordinary local variables or complete manifests in `_helpers.tpl`. A helper should return a small, stable fragment or value while resource templates remain readable.

## Build resource templates

Use this order:

1. a license header belonging to the target project;
2. the resource creation condition;
3. `apiVersion` and `kind`;
4. `metadata.name`, namespace, labels, and annotations;
5. `spec` in an order close to the Kubernetes API;
6. user extensions next to the fields they extend.

Do not copy Broadcom's copyright header into a new project. Use the target repository's owner and license.

Use capability helpers for API versions that vary across Kubernetes releases:

```yaml
apiVersion: {{ include "common.capabilities.deployment.apiVersion" . }}
kind: Deployment
```

Use `common.labels.standard` for resource metadata and pod templates. Use `common.labels.matchLabels` for immutable selectors. Keep workload selectors, pod labels, and Service selectors identical.

Add a component label outside the `common.labels.*` result when it distinguishes roles:

```yaml
labels: {{- include "common.labels.standard" (dict "customLabels" $podLabels "context" $) | nindent 4 }}
  app.kubernetes.io/component: replica
```

Do not add a component label to a single-component chart when it conveys no information and no existing selector requires it.

Render `commonAnnotations`, `podAnnotations`, Service annotations, affinity, node selectors, tolerations, topology spread constraints, extra environment variables, volumes, mounts, sidecars, and init containers with `common.tplvalues.render` when they support templated values.

Add checksums only for ConfigMaps and Secrets whose changes should roll the workload. Do not checksum a user-owned existing object without a deliberate `lookup` strategy.

Gate each optional resource with its matching value, such as `metrics.enabled`, `serviceAccount.create`, `persistence.enabled`, or `networkPolicy.enabled`. Do not combine unrelated features behind one generic `enabled` flag.

## Minimum functional review

For each workload, consider but do not mechanically add: image pull settings; command, args, and environment; service accounts; security contexts; resources; probes; scheduling; labels and annotations; lifecycle hooks; volumes; sidecars and init containers; update strategy, PDB, and autoscaling; Service, ingress, persistence, network policy, and metrics.

Support existing ConfigMaps or Secrets only with clear precedence over inline values. A smaller chart with a correct public API is better than a large chart full of dead options.

## Avoid these anti-patterns

- Do not wrap a single component in an artificial `app` block.
- Do not split values into `master` and `replica` when only one pod type is rendered.
- Do not keep component-specific settings at the root when components require independent configuration.
- Do not create variables at the beginning of a template "for later."
- Do not copy standard helpers from Helm scaffolding.
- Do not reimplement logic already available in Opsolving `common`.
- Do not create a helper merely to shorten one expression.
- Do not copy current Bitnami templates without checking compatibility with `common` 2.31.4.
- Do not use Bitnami's OCI repository for the `common` dependency.
- Do not generate resources that cannot be disabled or configured through documented values.
- Do not introduce two names for one option without a deprecation plan.
- Do not run `helm install`, `helm upgrade`, cluster tests, or Kubernetes connections for ordinary validation.

## Verify the result

Before finishing:

1. confirm that every `.Values` path used by a template exists in `values.yaml`;
2. confirm that every public value is consumed by a template or is an intentional global override;
3. confirm that every `common.*` helper exists in Opsolving `common` 2.31.4 and receives the correct dictionary shape;
4. confirm that `Chart.yaml` points to `https://opsolving.github.io/charts/`;
5. confirm that Service and workload selectors match pod labels;
6. confirm that local variables are defined as close as possible to their uses;
7. confirm that `_helpers.tpl` contains only chart-specific logic;
8. check YAML formatting, conflict markers, and the final diff;
9. run `helm lint` and `helm template` only when required dependencies are already available locally;
10. do not run `helm dependency update` without approval when environment rules require confirmation before fetching dependencies.

If dependencies are unavailable, perform static checks and report that Helm rendering was not run.

## References

When a design decision is unclear, inspect:

- `bitnami/redis/values.yaml`, `templates/master/`, `templates/replicas/`, `templates/sentinel/`, and `_helpers.tpl` for separate roles;
- `bitnami/redis-cluster/values.yaml`, `redis-statefulset.yaml`, and `_helpers.tpl` for one pod type with cluster topology;
- `bitnami/nginx/values.yaml`, `deployment.yaml`, and `svc.yaml` for a single component and root-level values;
- `bitnami/postgresql/values.yaml`, `templates/primary/`, and `templates/read/` for multiple components;
- `bitnami/flink/templates/_helpers.tpl` for a small component-specific helper set;
- `https://github.com/opsolving/charts/tree/main/opsolving/common` for the actual library API;
- `https://raw.githubusercontent.com/opsolving/charts/main/docs/index.yaml` for the published version and package URL.

Always derive the final chart from the deployment model of the target application. Use reference charts as examples of decisions, not as sources for unconditional copying.
