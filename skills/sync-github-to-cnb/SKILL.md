---
name: sync-github-to-cnb
description: Use when a project needs GitHub mirrored to CNB for China-friendly access, GitHub Actions to sync code/tags/releases, CNB Release asset upload, or split global/china update sources.
---

# Sync GitHub To CNB

## Overview

Use this skill to design or implement a reusable bridge from a GitHub repository to a CNB mirror. The goal is usually to keep GitHub as the main development and release source while making code, tags, release notes, and downloadable assets available from CNB for users who cannot reliably access GitHub.

Do not add project-specific habits such as "commit every local change" or "push after every edit" to this skill. Treat commit and release cadence as the caller project's policy.

## Fit Check

Use this pattern when:

- A project has users in mainland China who may have slow or blocked GitHub access.
- The app checks updates from Git tags, Release pages, or downloadable assets.
- GitHub remains the source of truth, but CNB should mirror code, tags, or release artifacts.
- The project needs separate global/china builds or update channels.

Do not use it when the user only needs a one-time manual upload, a private backup with no release/update flow, or when CNB should be the primary upstream instead of GitHub.

## Ask For Inputs

Before writing project-specific workflow files, collect only the missing values:

- GitHub repository URL, for example `https://github.com/OWNER/REPO`.
- CNB repository URL, for example `https://cnb.cool/OWNER/REPO.git`.
- CNB web Release page URL, for example `https://cnb.cool/OWNER/REPO/-/releases`.
- Secret name that stores the CNB token or password in GitHub Actions. Recommend `CNB_TOKEN` for new projects; adapt if the repo already uses another secret.
- Whether Release assets should sync automatically, manually by workflow dispatch, or both.
- Expected Release asset patterns, especially if the project publishes multiple channels such as `global` and `china`.

Never ask the user to paste tokens into the repository. Tell them to store credentials in GitHub Secrets.

## Recommended Architecture

Use two independent GitHub Actions workflows:

1. Code and tag mirror workflow, triggered by GitHub `push`.
2. Release sync workflow, triggered by GitHub Release `published`/`edited` and optional `workflow_dispatch`.

Keep these separate so normal code mirroring does not depend on Release asset upload, and Release retries can run without another code push.

```text
GitHub push
  -> sync repository contents to CNB
  -> sync missing Git tags to CNB

GitHub Release published or edited
  -> read Release title/body/tag
  -> download GitHub Release assets
  -> create or update matching CNB Release
  -> upload missing CNB Release assets
```

## Code And Tag Sync

A practical GitHub Actions pattern is:

- Checkout with full history: `fetch-depth: 0`.
- Mirror repository content to CNB using a git sync action/container, or plain `git push --mirror` if that is acceptable for the project.
- Sync tags carefully: read CNB's existing tags first and push only missing tags.

Avoid blindly pushing all tags if CNB may already contain tags. Existing tag objects can differ and cause the whole workflow to fail.

Minimal tag sync logic:

```bash
git ls-remote --tags --refs "$CNB_REPO_URL" \
  | awk '{print $2}' \
  | sed 's#refs/tags/##' > /tmp/cnb-tags.txt

while read -r tag; do
  [ -z "$tag" ] && continue
  if grep -Fxq "$tag" /tmp/cnb-tags.txt; then
    echo "Tag $tag already exists on CNB, skipping."
    continue
  fi
  git push "$CNB_REPO_URL" "refs/tags/$tag:refs/tags/$tag"
done < <(git tag --list)
```

When authentication is needed, use a GitHub Actions secret. Do not print token values in logs.

## Release Sync

Use a Release workflow when CNB should expose the same downloadable files as GitHub Releases.

Recommended workflow behavior:

- Trigger on `release: [published, edited]`.
- Add `workflow_dispatch` with a required `tag` input for manual retries.
- Get tag/title/body from the event, or from `gh release view <tag>` during manual dispatch.
- Download GitHub Release assets with `gh release download`.
- Call a script that talks to the CNB Release API.
- Skip CNB assets that already exist by file name.

For projects with assets generated shortly before Release publication, wait/retry until the expected asset count exists. Keep the retry bounded.

## CNB Release API Script Pattern

Use a script rather than embedding all API details in YAML. The script should:

- Accept `TagName`, `Title`, `Body` or `BodyFile`, `TargetCommitish`, `Slug`, `CnbUser`, `CnbToken`, and `Assets`.
- Read token from an environment variable such as `CNB_TOKEN` by default.
- Get CNB Release by tag.
- Create the CNB Release if missing; update title/body if it exists.
- For each asset, skip if an asset with the same file name already exists.
- Upload files through the CNB upload endpoint, then attach them to the Release.
- Print a final CNB Release URL, not secret values.

Keep API upload retries finite. If upload fails repeatedly, stop and give the user manual release instructions instead of looping.

## Global And China Update Sources

For apps that check updates, make the channel explicit at build time:

- `global`: update repo/page points to GitHub.
- `china`: update repo/page points to CNB.

A typical generated config contains:

```json
{
  "update_channel": "china",
  "update_repo_url": "https://cnb.cool/OWNER/REPO.git",
  "release_page_url": "https://cnb.cool/OWNER/REPO/-/releases",
  "release_api_url": ""
}
```

Prefer Git tag based update checks for the China channel, because GitHub API access may be unreliable for the target users. A Release API can remain as a fallback for the global channel.

## Safety Checklist

Before finishing an implementation, verify:

- Workflow files contain no plaintext tokens, passwords, cookies, or personal credentials.
- CNB URL, GitHub URL, owner, repo, and slug match the target project.
- Code/tag sync and Release sync are separate or intentionally combined.
- Tag sync skips existing CNB tags.
- Release sync can be retried manually by tag.
- CNB asset upload skips existing file names.
- Global/china build config does not include local user data.
- Documentation explains why CNB exists: improved access for users who cannot reliably reach GitHub.

## Common Mistakes

- Hard-coding a token in YAML or scripts instead of using GitHub Secrets.
- Assuming every project uses the same CNB owner/repo path as a previous project.
- Syncing Release metadata but forgetting downloadable assets.
- Mirroring code but forgetting tags, causing update checks by tag to fail.
- Writing the source project's ordinary commit/push policy into this reusable skill.
- Making CNB the update source for all users instead of keeping separate global/china channels when both audiences exist.
