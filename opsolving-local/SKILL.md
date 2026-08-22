---
name: opsolving-local
description: Work safely in a locally opened Git project or a wrapper workspace containing separate cloned repositories. Use when deciding whether the opened directory is itself the repository or only contains child repositories, especially when Git might walk upward into an unrelated parent .git; select the owning repository, confine work to it, and complete any explicitly authorized commit and push there.
---

# Opsolving - Local

## Goal

Work in the repository owned by the opened project. If the opened directory has its own `.git`, use that directory directly. Only treat the opened directory as a multi-repository workspace when it does not own `.git` and instead contains independent child repositories.

## Choose the operating mode

### Direct repository

When the opened project directory contains its own `.git` directory or worktree `.git` file:

- treat the opened directory as the default repository for the task;
- perform inspection, edits, tests, Git operations, commits, and pushes from that repository;
- do not reinterpret its immediate children as sibling repositories merely because they are directories;
- select a nested repository only when the user explicitly targets it or the requested files are clearly owned by it.

### Wrapper workspace

When the opened project directory does not contain its own `.git`:

- treat it as a workspace wrapper rather than a repository;
- treat each immediate child directory, such as `project/*`, as an independent project candidate;
- select the child repository that owns the requested change before editing or running Git mutations;
- do not fall back to a `.git` directory found above the wrapper.

## Repository boundary

- Prefer the opened directory's own `.git` whenever it exists and resolves back to that directory.
- Never use a `.git` directory found above the opened project or wrapper workspace. It belongs to unrelated local state and is outside the task.
- Never run Git mutations from a wrapper workspace. Set the working directory to the selected child repository first.
- Do not fall back to a parent repository when a child is not a Git repository. Stop and resolve the intended project instead.
- When a task spans several child repositories, handle each one as a separate working tree with its own instructions, status, diff, tests, commit, and push.

## Select and verify the project

1. Read the opened project's `AGENTS.md` when present.
2. Check whether the opened directory owns `.git` before interpreting it as a wrapper.
3. In direct-repository mode, select the opened directory. In wrapper mode, list its immediate children and select the child from the user's request and the files that own the behavior. If the target remains ambiguous after read-only inspection, ask before editing.
4. Read the selected repository's governing `AGENTS.md` files.
5. Verify the repository root without allowing upward discovery:

```bash
opened_root="$(pwd -P)"

if [ -e "${opened_root}/.git" ]; then
  project_dir="${opened_root}"
  git_ceiling="$(dirname "${opened_root}")"
else
  project_dir="${opened_root}/<selected-child>"
  git_ceiling="${opened_root}"
fi

GIT_CEILING_DIRECTORIES="${git_ceiling}" git -C "${project_dir}" rev-parse --show-toplevel
```

In direct-repository mode, the resolved root must equal the opened directory. In wrapper mode, it must be the selected child repository or an intentional repository nested inside it. A root above the opened project or inside a sibling project is invalid and must not be used.

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
