---
name: opsolving-local
description: Work in a local workspace whose immediate child directories are separate cloned Git repositories. Use when a task starts from a wrapper directory such as project/ and Git might otherwise walk upward into an unrelated parent .git; select the intended child repository, confine work to it, and complete any explicitly authorized commit and push there.
---

# Opsolving - Local

## Goal

Treat the current project directory as a workspace containing independent repositories, not as one flat repository. Work only in the child repository that owns the requested change, even when Git can discover a different `.git` directory higher in the filesystem.

## Repository boundary

- Treat each immediate child directory under the workspace, such as `project/*`, as an independent project candidate.
- Never use a `.git` directory found above the workspace root. It belongs to unrelated local state and is outside the task.
- Never run Git mutations from the workspace wrapper. Set the working directory to the selected child repository first.
- Do not fall back to a parent repository when a child is not a Git repository. Stop and resolve the intended project instead.
- When a task spans several child repositories, handle each one as a separate working tree with its own instructions, status, diff, tests, commit, and push.

## Select and verify the project

1. Read the workspace-level `AGENTS.md` when present, then list the immediate child directories.
2. Select the child project from the user's request and the files that actually own the behavior. If the target remains ambiguous after read-only inspection, ask before editing.
3. Read the selected child project's governing `AGENTS.md` files.
4. Verify the repository root without allowing upward discovery:

```bash
workspace_root="$(pwd -P)"
project_dir="${workspace_root}/<selected-child>"
GIT_CEILING_DIRECTORIES="${workspace_root}" git -C "${project_dir}" rev-parse --show-toplevel
```

The resolved root must be the selected child repository or an intentional repository nested inside it. A root above the workspace or inside a sibling project is invalid and must not be used.

## Make and verify changes

- Inspect the selected repository's current branch, remote, status, and relevant history before editing.
- Preserve unrelated tracked and untracked work. Modify only files owned by the selected project and requested task.
- Run the narrowest relevant tests or static checks from that repository.
- Review `git diff --check`, the complete scoped diff, and repository status before committing.
- Do not copy changes into a home-directory checkout or another clone merely because it contains similarly named files.

## Commit and push

When the current request or governing instructions explicitly authorize commit and push, both are required completion steps for every changed child repository:

1. stage only the intended files in that repository;
2. create a clear commit on its currently checked-out branch;
3. push that same branch to its configured remote;
4. fetch or otherwise verify that local `HEAD` matches the pushed remote branch and that the working tree is clean.

Do not treat automatic skill selection alone as authorization for Git mutations. Never force-push, rewrite history, reset unrelated work, or push through a parent repository. If the remote advanced, fetch and integrate it using the repository's policy; do not bypass the rejection with force.
