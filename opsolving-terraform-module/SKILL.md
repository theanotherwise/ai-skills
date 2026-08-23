---
name: opsolving-terraform-module
description: Create and refactor atomic reusable Terraform modules that correctly handle the complete documented configuration surface of each selected provider resource and use stable Opsolving file, collection, naming, output, example, and validation conventions. Use when building a provider-backed module or expanding one to full resource coverage. Do not use for Terragrunt live layouts, provider configuration, or applying infrastructure.
---

# Build complete Terraform modules

## Goal

Build atomic reusable child modules with a stable public API. By default, one module instance owns one core resource instance named `this` plus any repeatable subordinate resources that belong to that core. Do not turn the core resource into a fleet, batch, or `for_each` map unless the user explicitly requests a multi-core abstraction.

For every resource or data source intentionally owned by the module, support the complete configurable surface documented for the selected provider source, version, and release channel. Complete coverage applies within the selected resources; it does not authorize adding adjacent resources such as APIs, routers, NAT gateways, firewall rules, IAM bindings, monitoring, or deployment configuration that are outside the requested abstraction.

Do not reduce a resource to a convenient minimum. Every documented configurable argument and nested block must be either reachable through a typed module input or deliberately satisfied by an internal module invariant, such as binding a child resource directly to the module-owned parent ID. Correctly render and validate the resulting provider contract. Review computed-only attributes for outputs, but do not turn them into inputs or expose secrets merely to claim coverage.

## Required workflow

1. Read the governing repository instructions, existing module files, examples, tests, provider constraints, and lockfile policy.
2. Identify the one core resource, any subordinate resource collections, and the exact data sources the module owns. Keep unrelated provider resources outside the module and do not multiply the core merely because Terraform supports `for_each`.
3. Identify the provider source, minimum supported version, and stable or beta channel. Never silently switch channels.
4. Before adding or substantially changing a resource, read [references/resource-coverage.md](references/resource-coverage.md) and audit the official versioned provider documentation for that resource.
5. Design the complete typed input API before writing resource blocks. Preserve the target repository's existing compatible API when refactoring.
6. Implement the file, collection, resource-address, output, and example conventions below.
7. Add focused tests for the public contract and each meaningful optional or mutually exclusive path when the target repository supports Terraform tests.
8. Run the narrowest non-mutating validation available and report any check blocked by unavailable dependencies.

## File convention

Use lowercase filenames and group code by resource domain:

```text
versions.tf
locals.tf                    # only when real transformations exist
outputs.tf
v-<domain>.tf                # variables for the domain
d-<domain>.tf                # data sources, only when needed
r-<domain>.tf                # managed resources for the domain
examples/<name>/main.tf      # direct module call only
tests/<name>.tftest.hcl      # focused module tests
```

Keep related variable and resource filenames paired, for example `v-network.tf` and `r-network.tf`. Split a file only when the resulting domains remain coherent; do not create one file per trivial block.

Do not add `main.tf`, `providers.tf`, `backend.tf`, `terraform.tfvars`, or `imports.tf` to a reusable module under this convention. Add `moved.tf` only for an actual state-address migration. Do not keep empty `locals.tf`, `d-*.tf`, examples, or tests as placeholders.

Put only `required_version` and `required_providers` in `versions.tf`. Declare the minimum Terraform and provider versions required by the features used. Do not configure a provider or backend inside the child module; the caller owns provider configuration, credentials, aliases, and state.

## Stable collections and resource addresses

Choose collection types by identity and ordering semantics:

- Use one `object(...)` for the core resource configuration and render one core resource named `this`.
- Use `map(object(...))` with stable semantic keys and `for_each` for independently managed resource instances.
- Use `set(string)` or another primitive set for unordered unique primitive identities.
- Use `list(...)` only when ordering is part of the domain contract or the provider API explicitly requires an ordered sequence.
- Do not use `list(object(...))` with `count` for independently managed resources. Inserting or reordering an element must not shift resource addresses.
- Avoid `set(object(...))` for managed resource identity because changing any hashed field can change membership. Prefer an explicit stable map key.

Do not use `map(object(...))` for the core resource merely to make the module capable of creating an arbitrary number of cores. A module that owns one VPC and its subnetworks creates one network and a keyed subnetwork collection; a fleet of VPCs is a different abstraction and requires an explicit request.

The map key is the Terraform identity and must remain stable when mutable attributes change. Use the same key in outputs so consumers can select `output_map["semantic-key"]`. Do not derive identity from a list index, generated timestamp, provider-computed value, or another attribute that is unknown before apply.

A repeatable nested provider block may use `list(object(...))` when its order is meaningful to that one resource. If order is irrelevant but the provider requires a sequence, accept a stable map or set-shaped input and transform it deterministically at the rendering boundary.

## Names and resource labels

Use `snake_case` for Terraform variables, locals, resource labels, and outputs. Use lowercase hyphen-case only for filenames and directories.

Name a single instance of a provider resource `this`:

```hcl
resource "google_compute_network" "this" {}
```

Also use `this` for a `for_each` collection when the keys identify instances:

```hcl
resource "google_compute_subnetwork" "this" {
  for_each = var.subnetworks
}
```

When several fixed instances of the same resource type have different roles, use stable role labels such as `internal`, `external`, `primary`, or `replica`. Do not repeat the provider resource type in the local label.

## Input contract

Give every variable an explicit type and useful description. Use `nullable = false` for required values and collections that must not accept null. Represent optional provider arguments with stable optional attributes or nullable values; normally use `null` to omit an argument and preserve the provider's default rather than duplicating that default in the module.

Model nested blocks with explicit object types. Do not use `any`, arbitrary maps, JSON strings, or raw pass-through HCL to avoid defining the documented schema. Preserve value types across releases and use validation or preconditions for documented enums, ranges, formats, conflicts, required-together relationships, and mutually exclusive choices.

Do not expose an input that no resource or data source consumes. Do not use locals merely to rename variables. Locals should normalize complete resource inputs, derive stable names, or perform a transformation used in more than one place.

Bind structural relationships internally. When a subordinate resource must reference the core resource, assign the provider argument directly from the core resource, for example `network = google_compute_network.this.id`. Do not expose artificial selectors such as `network_key`, duplicate direct-reference alternatives, or parent project inputs when the module invariant already determines those values. Supporting the provider argument internally counts as complete coverage.

## Resource coverage

Every selected resource must pass the coverage audit in [references/resource-coverage.md](references/resource-coverage.md). The audit must distinguish caller-configurable arguments from arguments intentionally bound by the atomic module structure. Render all supported optional nested blocks conditionally and preserve provider distinctions between omitted, empty, false, zero, and an explicitly empty collection.

Do not hide unsupported combinations with permissive types and let the provider fail late. Encode documented constraints as early as Terraform permits. Do not add lifecycle workarounds, `ignore_changes`, `prevent_destroy`, API enablement, imports, or implicit dependencies merely because the provider documentation mentions operational behavior; add them only when the user or module contract requires them.

## Outputs and composition

Design outputs for callers and Terragrunt dependencies rather than mirroring file layout. Return scalar IDs, names, and provider references for the single core resource. For subordinate collections, return maps keyed by the same stable input keys:

```hcl
output "subnetwork_ids" {
  value = {
    for key, subnet in google_compute_subnetwork.this :
    key => subnet.id
  }
}
```

Expose additional computed attributes when a realistic consumer needs them or when the module contract promises full discovery. Mark sensitive outputs as sensitive and never expose write-only values or credentials. Avoid exporting the entire raw resource object as a shortcut for designing a stable API.

Keep module composition flat. A caller should pass another module's explicit output into this module's input instead of this module discovering remote state, hardcoding Terragrunt paths, or creating unrelated dependencies internally.

## Examples

Keep each ordinary example in one `examples/<name>/main.tf` file containing a direct module call with representative literal values. Do not reproduce the module's `v-*.tf`, `locals.tf`, `versions.tf`, provider configuration, or outputs inside the example unless the user explicitly requests a runnable root configuration with those responsibilities.

The basic example demonstrates the normal path; tests or additional named examples cover materially different modes. Do not overload one example with every optional argument merely to prove coverage.

## Validation

Run `terraform fmt -check -recursive` after formatting. Run `terraform validate` and `terraform test` when their providers and modules are already available or dependency initialization is authorized. Prefer mocked provider tests for module logic when the selected Terraform version supports them.

Do not run `terraform init` when it would fetch dependencies without authorization. Do not run `plan` against a real backend, `apply`, import, state mutation, provider authentication, or cloud API changes unless explicitly requested. Finish by reviewing the complete diff, checking conflict markers and repository status, and stating exactly which validation did and did not run.
