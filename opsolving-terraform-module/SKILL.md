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
7. Build a test-coverage matrix that maps every public input path, conditional nested block, documented constraint, stable collection key, and output path to a concrete test and assertion.
8. Run the complete non-mutating validation required by the module contract. If a provider or module dependency is unavailable, report the result as blocked and do not call the module complete or compliant.

## File convention

Use lowercase filenames in a flat module root and pair files by individual provider resource:

```text
versions.tf
locals.tf                    # only when real transformations exist
outputs.tf
v-<resource>.tf              # public inputs owned by one managed resource
d-<data-source>.tf           # exactly one top-level data block
r-<resource>.tf              # exactly one top-level resource block
examples/<name>/main.tf      # direct module call only
tests/<name>.tftest.hcl      # focused module tests
```

Every top-level managed `resource` block must have its own `r-<resource>.tf` file. Never place two different resource blocks, two fixed instances, or a core resource and a subordinate resource in the same `r-*.tf` file. Nested provider blocks and Terraform `dynamic` blocks belong inside their owning resource file and do not count as separate resources.

Give every configurable resource a matching `v-<resource>.tf` file containing only the public variables owned by that resource. Prefer one typed object for a single resource and one `map(object(...))` for a repeatable subordinate resource collection. Do not place inputs for another resource in the same variable file. Bind inherited core values internally instead of duplicating them in the subordinate variable contract.

For example, a module with one core service and repeatable endpoints uses `v-service.tf` with `r-service.tf` and `v-endpoint.tf` with `r-endpoint.tf`. It must not place both provider resource blocks in `r-service.tf`, and it must not place both public resource contracts in `v-service.tf`.

Keep resource files at the module root. Do not create a directory per provider resource merely to satisfy this separation. When several fixed instances of the same provider type are required, give each resource block a role-qualified pair such as `v-address-internal.tf` and `r-address-internal.tf`.

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

Do not use `map(object(...))` for the core resource merely to make the module capable of creating an arbitrary number of cores. A module that owns one core resource and its children creates one core instance and a keyed child collection; a fleet of core resources is a different abstraction and requires an explicit request.

The map key is the Terraform identity and must remain stable when mutable attributes change. Use the same key in outputs so consumers can select `output_map["semantic-key"]`. Do not derive identity from a list index, generated timestamp, provider-computed value, or another attribute that is unknown before apply.

A repeatable nested provider block may use `list(object(...))` when its order is meaningful to that one resource. If order is irrelevant but the provider requires a sequence, accept a stable map or set-shaped input and transform it deterministically at the rendering boundary.

## Names and resource labels

Use `snake_case` for Terraform variables, locals, resource labels, and outputs. Use lowercase hyphen-case only for filenames and directories.

Name a single instance of a provider resource `this`:

```hcl
resource "example_service" "this" {}
```

Also use `this` for a `for_each` collection when the keys identify instances:

```hcl
resource "example_endpoint" "this" {
  for_each = var.endpoints
}
```

When several fixed instances of the same resource type have different roles, use stable role labels such as `internal`, `external`, `primary`, or `replica`. Do not repeat the provider resource type in the local label.

## Input contract

Give every variable an explicit type and useful description. Use `nullable = false` for required values and collections that must not accept null. Represent optional provider arguments with stable optional attributes or nullable values; normally use `null` to omit an argument and preserve the provider's default rather than duplicating that default in the module.

Model nested blocks with explicit object types. Do not use `any`, arbitrary maps, JSON strings, or raw pass-through HCL to avoid defining the documented schema. Preserve value types across releases and use validation or preconditions for documented enums, ranges, formats, conflicts, required-together relationships, and mutually exclusive choices.

Do not expose an input that no resource or data source consumes. Do not use locals merely to rename variables. Locals should normalize complete resource inputs, derive stable names, or perform a transformation used in more than one place.

Bind structural relationships internally. When a subordinate resource must reference the core resource, assign the provider argument directly from the core resource, for example `parent_id = example_service.this.id`. Do not expose artificial selectors such as `parent_key`, duplicate direct-reference alternatives, or inherited scope inputs when the module invariant already determines those values. Supporting the provider argument internally counts as complete coverage.

## Resource coverage

Every selected resource must pass the coverage audit in [references/resource-coverage.md](references/resource-coverage.md). The audit must distinguish caller-configurable arguments from arguments intentionally bound by the atomic module structure. Render all supported optional nested blocks conditionally and preserve provider distinctions between omitted, empty, false, zero, and an explicitly empty collection.

Do not hide unsupported combinations with permissive types and let the provider fail late. Encode documented constraints as early as Terraform permits. Do not add lifecycle workarounds, `ignore_changes`, `prevent_destroy`, API enablement, imports, or implicit dependencies merely because the provider documentation mentions operational behavior; add them only when the user or module contract requires them.

## Outputs and composition

Design outputs for callers and Terragrunt dependencies rather than mirroring file layout. Return scalar IDs, names, and provider references for the single core resource. For subordinate collections, return maps keyed by the same stable input keys:

```hcl
output "endpoint_ids" {
  value = {
    for key, endpoint in example_endpoint.this :
    key => endpoint.id
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

Before finishing, inspect top-level declarations and confirm that every `r-*.tf` contains exactly one managed resource block, every `d-*.tf` contains exactly one data block, and each configurable resource has the correctly named `v-*.tf` counterpart without variables belonging to another resource.

Require zero uncovered rows in the test-coverage matrix:

- every public input and nested input path has a positive test that asserts the value reaches the intended provider argument or internal binding;
- every documented enum, range, format, conflict, required-together rule, and mutually exclusive choice has a negative test;
- every optional nested block has tests for presence and omission, including empty-collection behavior when it differs from omission;
- every stable collection verifies its resource keys and corresponding output keys without relying on list indexes;
- every public output has an assertion for its shape, keys, sensitivity where relevant, and a representative value;
- a test that only supplies a value without asserting the rendered resource or output does not count as coverage.

Use provider documentation to derive tests; do not hardcode a provider-specific checklist into the skill. When computed values are unknown during `plan`, use provider mocks or mock defaults appropriate to the selected Terraform version so output assertions remain deterministic.

`terraform fmt` and static declaration checks do not replace `terraform validate` and the required test suite. If those checks cannot run because dependencies are unavailable and initialization is not authorized, identify the exact blocked commands and hand off the module as unverified rather than complete.

Do not run `terraform init` when it would fetch dependencies without authorization. Do not run `plan` against a real backend, `apply`, import, state mutation, provider authentication, or cloud API changes unless explicitly requested. Finish by reviewing the complete diff, checking conflict markers and repository status, and stating exactly which validation did and did not run.
