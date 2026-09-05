---
name: opsolving-release
description: Release or deploy Git-backed projects through the repository's established tag workflow. Use when the user asks to release, deploy, publish, ship, "make a release", "do a release", "wdrozyc", or "wdrazaj"; inspect existing local and remote release tags, create and push the analogous next tag, and verify the tag-triggered build with the Actions MCP server when it is available. Do not use for an ordinary commit or push request that does not ask for a release or deployment.
---

# Opsolving - Release

## Release contract

For this workflow, a release is a new Git tag on the exact release commit, pushed to the repository's configured remote. Treat an explicit request to release or deploy as authorization to commit the requested release changes when needed, push that commit's current branch, create the release tag, and push the tag. Do not ask for separate confirmation for those Git operations.

A commit or branch push alone is not a release. Do not create a tag for an ordinary commit or push request that does not include release or deployment intent. Do not create a hosting-provider Release object unless the user explicitly asks for one.

## Required workflow

1. Read the governing project instructions and identify every repository that owns requested changes. Handle each repository independently.
2. Inspect the current branch, remotes, status, complete release diff, recent history, and any repository release documentation. Preserve unrelated work and stage only files belonging to the requested release.
3. Inspect both local and remote tags before choosing a name. Do not rely on a stale local tag list. Review enough recent release tags and their target commits to identify the relevant tag family, ordering, separators, prefixes, version width, timestamp and hash fields, and whether tags are annotated or lightweight.
4. Derive the next tag by continuing the established repository pattern. Use the repository's release rules when they exist. Never replace an established format with a preferred convention.
5. If the next tag is deterministic, proceed without asking. For semantic versions, use the bump requested by the user or required by repository policy; if neither determines major, minor, or patch, ask for that choice. If there are no prior release tags, several incompatible active tag families, or any other ambiguity that changes the tag name, ask the user rather than inventing a format.
6. Run the narrowest relevant non-mutating checks required by the project. Review the final diff and status, then commit pending release changes with a clear message when a release commit is needed.
7. Confirm that the intended tag does not already exist locally or remotely and that the release commit is the expected `HEAD`. Push the current branch when it contains the release commit, create the tag using the repository's established tag type and message style, and push that exact tag. Never force-push, move an existing tag, or retag a different commit.
8. Verify that the remote branch contains the release commit when a branch push was required, and that the remote tag resolves to the same commit as the local tag.
9. When the Actions MCP server is available, use it to find the workflow or build triggered by the pushed tag. Follow the relevant run to a terminal result, inspect failed jobs or logs when needed, and report the build conclusion. A queued or running workflow is not a successful build. If no matching run appears, report that fact instead of claiming success or triggering an unrelated workflow.

## Tag pattern rules

- Continue monotonic integer tags such as `v17` with the next integer while preserving the prefix and padding.
- Preserve semantic-version prefixes, separators, prerelease suffixes, and repository-specific bump rules.
- For timestamp-and-commit tags, generate the timestamp at release time in the repository's established timezone and layout, and use the release commit hash with the same abbreviation length.
- In a monorepo or repository with component tags, select the tag family for the component being released and preserve its component prefix.
- Match the existing annotated or lightweight tag form. When annotated tags are used, follow the established message style.

## Completion evidence

Report the repository, release commit, exact tag, pushed branch and remote, remote tag verification, checks run, and Actions build result when available. State separately whether runtime deployment was verified; a successful tag build proves the release pipeline completed only to the stage shown by that workflow.
