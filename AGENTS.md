# AI Skills Repository Instructions

## Purpose

This repository stores the Homearo promotional-images Codex skill. Keep it focused, self-contained, and suitable for discovery by its frontmatter metadata.

## Structure

The only retained skill directory is `homearo-promotional-images`; its name matches the `name` field in its `SKILL.md`. Optional UI metadata belongs in `agents/openai.yaml`; skill-specific references, scripts, and assets belong in their standard subdirectories only when needed. Airflow, Python API, Vue 3, Bitnami-style Helm and Terraform implementation packages are maintained in `actions-api/src/app/mcp/skills` and served at `https://mcp.psem.io/mcp/skills`; update those packages in the MCP repository.

`homearo-promotional-images` contains Polish instructions for narrative Homearo promotional graphics and captions, including prominent typography, both mandatory promotional messages, and a uniform operator signature. Each publication uses a separately generated image and distinct caption consisting of one 15–20-word sentence (excluding the address, separator, emoji, and hashtags) in the format `www.homearo.pl - Description.`, without @ mentions or parenthesized account names. The sentence tells the same story as the image. Caption layout is mandatory on every platform: LinkedIn, Instagram, and Facebook require one blank line between the sentence and hashtags. Paste the complete caption once as plain text, leave address and hashtag parsing to the platform, and save or publish within the authorized scope without adding profile mentions. Emoji are optional on Instagram/Facebook and excluded on LinkedIn; captions on platforms supporting hashtags require at least five distinct real-estate-related tags, also reflecting the image subject. This also applies to separate Homearo page and Mateusz Adam Katana profile variants. Its `references/instagram-style.md` maps composition directions from the historical Instagram series; the source images belong to the Homearo workspace, not this repository. Save generated campaign artifacts in the target project rather than the skill directory.

## Workflow

Validate every added or changed skill with the skill-creator `quick_validate.py` script. Keep the root `README.md` list of available skills accurate when repository contents change.

## Constraints

Do not add auxiliary README, changelog, installation guide, or quick-reference files inside a skill directory. Do not fetch dependencies, contact clusters, deploy workloads, or start services during skill validation unless explicitly requested.
