# Complete provider resource coverage

Read this reference before implementing a new provider resource or expanding an existing resource wrapper. Its purpose is to prevent a module from supporting only the easiest arguments while silently excluding documented capabilities.

## Establish the documentation boundary

Record the atomic module boundary: the one core resource instance, its subordinate resource collections, the exact provider source, minimum supported version, selected stable or beta channel, and every resource or data source owned by the module. Use this precedence:

1. user requirements and governing repository instructions;
2. official documentation for the selected provider version and channel;
3. an already available `terraform providers schema -json` result for the selected provider;
4. official provider source code or upstream service API documentation when the provider documentation is ambiguous.

Do not inspect only the provider's latest documentation when the module supports an older minimum version. Implement the intersection promised by the module's provider constraint, or raise the minimum version deliberately when a requested capability requires it. Do not fetch or install a provider merely to inspect its schema unless dependency fetching is authorized.

Full coverage is bounded by the selected resource, but it does not require every provider argument to become a public input. A structural argument satisfied by the module abstraction, such as a subnetwork's `network` pointing directly to the module-owned VPC ID, is fully handled by an internal binding. It must not be duplicated as an artificial selector input.

For example, an atomic VPC module owns one `google_compute_network.this` and may own a keyed collection of `google_compute_subnetwork.this`. It must handle all remaining configurable arguments and nested blocks supported by those resource versions; it does not create a map of core networks or expose `network_key`, and it does not need Cloud Router, Cloud NAT, firewall, IAM, or API-enablement resources unless those resources are explicitly in scope.

## Build a coverage checklist

Use a temporary checklist with one row for every documented path:

```text
documentation path | kind | source: input/internal | default/omission | rendering | validation | test
```

Include:

- required and optional arguments;
- single and repeatable nested blocks;
- arguments that are optional and computed;
- deprecated but still supported configurable arguments;
- conflicts, exactly-one-of, at-least-one-of, required-with, and mutually exclusive groups;
- documented enum values, ranges, formats, and collection limits;
- provider distinctions between omitted, null, empty, false, and zero;
- ForceNew or replacement-sensitive arguments;
- sensitive, write-only, and ephemeral arguments;
- computed-only attributes and import identifiers for review, even when they do not become module inputs.

Keep the checklist as a working artifact unless the target repository explicitly tracks coverage documentation. Completion requires every configurable row to be implemented as a typed input, satisfied by a clear internal invariant, or given a deliberate user-approved exclusion. Do not silently omit a row because it is uncommon, advanced, or difficult to model.

## Map documentation into the module API

Use these mappings unless the provider contract requires something different:

| Provider schema | Module representation |
|---|---|
| Required scalar argument | Required typed variable or required object attribute |
| Optional scalar argument | Optional typed attribute or nullable value, usually omitted with `null` |
| Single optional nested block | Optional typed object rendered conditionally |
| Repeatable independently identified items | `map(object(...))` with stable keys |
| Unordered unique primitive items | `set(<primitive>)` |
| Ordered provider sequence | `list(...)` because order is part of the contract |
| Reference to the module-owned core or parent | Direct resource attribute binding, not a selector input |
| Computed-only attribute | No input; consider an explicit consumer-oriented output |
| Sensitive or write-only argument | Sensitive input; never return the write-only value |
| Mutually exclusive alternatives | Typed optional alternatives plus validation or preconditions |

Do not use `any` to mirror the provider schema. Explicit types make completeness, compatibility, and errors reviewable. Preserve provider defaults by omission unless the module intentionally guarantees a different default. An optional block that is absent must not render an empty block unless the provider defines those states as equivalent.

Do not expose both an internal key and a direct external reference for the same structural relationship. If the module owns the parent, bind the child directly to it. Supporting children of an externally owned parent is a different abstraction and should be a separate module or an explicitly requested mode, not an accidental branch in the same input object.

When a resource supports several operational modes, keep one coherent resource input contract with validated alternatives. Do not create several partial wrappers that each expose a different subset of the same underlying resource unless they represent genuinely distinct abstractions requested by the user.

## Review computed attributes and outputs

Review every documented computed attribute, but expose only stable, safe outputs that form a useful module contract. IDs, names, self-links, endpoints, and status identifiers commonly matter to consumers. Return scalar outputs for the single core resource and collection outputs as maps keyed by the same stable keys used by subordinate `for_each` resources.

Do not export an entire provider resource object merely to claim full computed coverage. That couples consumers to provider schema changes and can expose sensitive values. If the user explicitly requires every safe computed attribute, return a documented typed object and omit sensitive, write-only, transient, or meaningless implementation fields.

## Test the coverage

The basic example should exercise the normal configuration, not every switch. Use focused Terraform tests for:

- required arguments and provider-default omission;
- each optional nested block or materially different mode;
- stable `for_each` keys and output keys;
- invalid enums, ranges, formats, and mutually exclusive combinations;
- explicit empty collections when they differ from omission;
- sensitive output behavior where applicable.

Use mocked providers for module-logic tests when supported and already initialized. Mocking does not prove remote API behavior, so use provider documentation and existing integration tests for API-specific semantics. Do not create real infrastructure for ordinary skill validation.

## Provider upgrades

When the provider constraint changes, repeat the audit for every owned resource. Identify added, removed, deprecated, renamed, or type-changed arguments and nested blocks. Update the input API, rendering, validations, examples, tests, and minimum Terraform version together; do not bump the version while leaving the module's documented resource surface stale.
