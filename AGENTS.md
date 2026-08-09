# AI Skills Repository Instructions

## Purpose

This repository stores reusable Codex skills. Keep each skill focused, self-contained, and suitable for discovery by its frontmatter metadata.

## Structure

Each top-level skill directory must use the same lowercase hyphen-case name as the `name` field in its `SKILL.md`. Optional UI metadata belongs in `agents/openai.yaml`; skill-specific references, scripts, and assets belong in their standard subdirectories only when needed.

## Workflow

Validate every added or changed skill with the skill-creator `quick_validate.py` script. Keep the root `README.md` list of available skills accurate when repository contents change.

## Constraints

Do not add auxiliary README, changelog, installation guide, or quick-reference files inside a skill directory. Do not fetch dependencies, contact clusters, deploy workloads, or start services during skill validation unless explicitly requested.
