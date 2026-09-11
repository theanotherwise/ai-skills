---
name: opsolving-docs-ai-context
description: Read and maintain project-wide technical documentation, business assumptions, and working procedures in a project-name-docs-ai-context repository. Use when understanding or changing a project that keeps shared context in such a repository, including keeping that documentation aligned with implementation changes.
---

# Opsolving - Docs AI Context

## Project documentation repository

The project or workspace root may contain a directory named `<project-name>-docs-ai-context` that is its own Git repository. This repository holds the shared documentation for the whole project: what it does, its business goals and assumptions, technical design, component responsibilities, and how work should be carried out. Read the relevant context before planning or implementing changes, and keep it current as part of the same task.

For example, independent repositories may share one workspace:

```text
project/
|-- project-docs-ai-context/
|-- project-api/
`-- project-front/
```

## Find and read the context

1. Read the governing project instructions and identify the project or workspace root. Look for the matching `<project-name>-docs-ai-context` repository there; when working inside a component repository, also check its known containing workspace for that sibling repository. Confirm ownership from the instructions and documentation rather than selecting an unrelated repository merely because its name has the same suffix.
2. Read the documentation repository's own `AGENTS.md` and existing documentation entry points or indexes. Follow the relevant links and search for the components, business processes, and procedures affected by the task. Use its existing organization; filenames such as `DIRECTORY_STRUCTURE.md` or `MICROSERVICES.md` are examples, not required files.
3. If no matching repository exists, use the project's existing documentation and instructions. Do not create or clone a documentation repository merely because this skill mentions the convention. If several candidates remain plausible, clarify ownership before editing them.

The relevant context can include business requirements and terminology, user journeys and rules, architectural decisions and their rationale, repository boundaries, data models, API contracts, integrations, configuration, environments, and development, testing, release, deployment, and operational procedures. Read enough to understand the task's constraints without loading unrelated documentation.

Distinguish implemented behavior from planned work, assumptions, and open decisions. If implementation and documentation disagree, investigate the discrepancy against the user's request and available evidence. Do not silently redefine a business requirement to match the current code or treat an unimplemented proposal as an existing feature.

## Keep documentation aligned with changes

Treat updates to the affected documentation as part of completing the requested change. Respect an explicit read-only or repository-only scope; when it excludes documentation edits, identify the necessary follow-up instead of modifying an excluded repository. Documentation procedures do not independently authorize deployments or other external mutations.

- Update the relevant technical descriptions when architecture, responsibilities, interfaces, data, configuration, dependencies, or execution behavior change.
- Update business assumptions, rules, workflows, and constraints when the requested change affects them. Record agreed decisions and their rationale; keep unresolved decisions explicitly unresolved.
- Update applicable setup, development, testing, release, and operational instructions, including commands, examples, diagrams, and links that the change makes inaccurate.
- Edit the existing source sections and indexes rather than appending a conversation transcript or duplicating the same documentation across component repositories. Preserve the repository's language and organization; add a focused document only when the existing structure needs it.

For example, adding an order approval stage may require updating the business workflow, state model, API contract, and operational procedure in `<project-name>-docs-ai-context` alongside changes in the owning application repositories.

Review documentation impact for every change. If no documented behavior, assumption, or procedure changes, leave accurate documentation intact and briefly state why no update was needed.

## Repository handling and verification

Handle the documentation repository independently from application and infrastructure repositories: read its instructions, verify its Git root, inspect its status and diff, and preserve unrelated work before editing. Follow the governing rules for required `AGENTS.md` files. Keep each repository's Git operations scoped to that repository and perform commits or pushes only when authorized.

Review the documentation against the final implementation diff, check affected paths, links, commands, and examples, and run the narrowest available non-mutating documentation checks. Report which context documents were read or updated and any remaining discrepancies or unverified procedures. Do not run documented deployment commands merely to verify the text.
