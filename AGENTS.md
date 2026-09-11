# AI Skills Repository Instructions

## Purpose

This repository stores reusable Codex skills. Keep each skill focused, self-contained, and suitable for discovery by its frontmatter metadata.

## Structure

Each top-level skill directory must use the same lowercase hyphen-case name as the `name` field in its `SKILL.md`. Optional UI metadata belongs in `agents/openai.yaml`; skill-specific references, scripts, and assets belong in their standard subdirectories only when needed.

`opsolving-docs-ai-context` covers discovering and maintaining the separate `<project-name>-docs-ai-context` repository that holds shared technical documentation, business assumptions, and working procedures.

`homearo-promotional-images` contains Polish instructions for narrative Homearo promotional graphics and captions, including prominent typography, both mandatory promotional messages, and a uniform operator signature. Each publication uses a separately generated image and distinct caption beginning with the portal URL on the same line, including separate Homearo page and Mateusz Adam Katana profile variants on LinkedIn and Facebook. Its `references/instagram-style.md` maps composition directions from the historical Instagram series; the source images belong to the Homearo workspace, not this repository. Save generated campaign artifacts in the target project rather than the skill directory.

## Workflow

Validate every added or changed skill with the skill-creator `quick_validate.py` script. Keep the root `README.md` list of available skills accurate when repository contents change.

## Constraints

Do not add auxiliary README, changelog, installation guide, or quick-reference files inside a skill directory. Do not fetch dependencies, contact clusters, deploy workloads, or start services during skill validation unless explicitly requested.
