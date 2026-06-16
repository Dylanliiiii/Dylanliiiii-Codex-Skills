---
name: sync-personal-skills-repo
description: Use when the user creates or edits a local Codex skill and wants to sync it to their personal GitHub skills repository after an explicit confirmation step.
---

# Sync Personal Skills Repo

## Overview

Use this skill to publish or update personal Codex skills in the user's dedicated GitHub archive. The workflow is semi-automatic: prepare, validate, summarize, ask for confirmation, then commit and push.

This skill is only about synchronizing skill folders. Do not apply it to ordinary project code, release packages, or unrelated documentation.

## Repository

- GitHub repo: `https://github.com/Dylanliiiii/Dylanliiiii-Codex-Skills.git`
- Default branch: `main`
- Store skills under `skills/<skill-name>/`.
- Preserve each skill's internal structure, especially `SKILL.md` and optional `agents/openai.yaml`, `scripts/`, `references/`, or `assets/`.

Recommended repo shape:

```text
Dylanliiiii-Codex-Skills/
  README.md
  skills/
    skill-name/
      SKILL.md
      agents/
        openai.yaml
```

## Trigger

Use this when the user says things like:

- "把这个 skill 同步到我的 skills 仓库。"
- "创建完 skill 后放到我的 GitHub skills repo。"
- "更新一下我个人 skills 仓库里的这个 skill。"
- "以后新建或修改 skill 后，确认再同步到仓库。"

If the user merely asks how a skill works, do not sync anything.

## Workflow

1. Identify the local skill folder. Prefer `C:/Users/pc/.codex/skills/<skill-name>` unless the user gives another path.
2. Read `SKILL.md` and any `agents/openai.yaml` to understand what will be published.
3. Validate the skill with `quick_validate.py` in UTF-8 mode.
4. Clone or update the personal skills repository in a temporary or user-approved workspace.
5. Copy the local skill folder to `skills/<skill-name>/` in the repository.
6. Exclude caches and generated noise: `__pycache__/`, `.pyc`, `.DS_Store`, temporary files, local logs, secrets, tokens, and private project data.
7. Inspect `git diff --stat` and the relevant diff summary.
8. Ask the user for explicit confirmation before commit/push unless the user already gave direct approval in the same turn.
9. Commit with a focused message such as `docs: sync <skill-name> skill`.
10. Push to `origin main`.

## Confirmation Rule

This repository is an archive of reusable global skills, so keep a human confirmation point before publishing.

If the user already says "同步并推送" or gives equivalent explicit approval for the specific skill in the current turn, that counts as confirmation. Otherwise, summarize the planned changes and ask before `git commit` or `git push`.

Never silently push a skill that may contain local paths, credentials, project-private details, or unfinished instructions.

## Safety Checks

Before publishing, verify:

- `SKILL.md` has valid YAML frontmatter with `name` and `description`.
- No placeholder text remains, such as `TODO` from a template.
- No tokens, cookies, passwords, personal access tokens, or private keys appear.
- No unrelated project-specific commit/push/release policy is included unless the skill is explicitly about that project.
- Absolute local paths are avoided unless they describe this user's personal Codex skill directory or are necessary for this personal workflow.
- `agents/openai.yaml`, if present, matches the skill purpose.
- The target repo diff only touches the intended `skills/<skill-name>/` folder and intentional repo-level docs.

## Handling Existing Skills

When the target skill already exists in the repository:

- Replace or update only that skill folder.
- Preserve files that still exist locally.
- Remove stale files only when they no longer belong to the current local skill.
- Show the user what changed before publishing when approval has not already been given.

## Common Mistakes

- Treating this as a generic backup workflow for any project files.
- Pushing without confirmation after a skill edit.
- Publishing caches or temporary files.
- Forgetting to validate the skill before copying it into the repository.
- Mixing multiple unrelated skills in one commit unless the user asked to sync them together.
