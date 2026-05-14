---
name: gitlab-create-mr
description: Create GitLab merge requests for repositories hosted on gitlab.tomatom.cz using the bundled gitlab-mr helper. Use this skill whenever the user asks Codex to create, open, publish, submit, or prepare a GitLab MR/merge request from a local branch after code changes are committed. This is for self-hosted GitLab merge requests, not GitHub pull requests or code-review-only tasks.
---

# GitLab Create MR

## Overview

Use this skill's bundled `scripts/gitlab-mr` helper to push the current branch
and create a GitLab merge request.

This skill is intended for Codex/agent workflows. It does not require a global
`gitlab-mr` command to be installed.

Resolve the helper path relative to this installed skill directory, not relative
to the user's project repository. For example, if this file is installed at
`<skill-dir>/SKILL.md`, run `<skill-dir>/scripts/gitlab-mr`.

## Workflow

1. Confirm the repo state before creating an MR:
   - run `git status --short --branch`
   - ensure the intended changes are already committed; the helper publishes committed branch state only
   - local uncommitted or untracked files are not included in the MR
2. Build a concise MR title and description from the actual change and verification performed.
3. Run a dry run first with the bundled helper:
   - `<skill-dir>/scripts/gitlab-mr --dry-run --title "<title>" --description "<summary and test notes>"`
4. If the dry run is correct, run the real command:
   - `<skill-dir>/scripts/gitlab-mr --title "<title>" --description "<summary and test notes>"`
5. Report the returned GitLab MR URL.

## Command Options

Use these only when they match the user's request or repo conventions:

- `--target <branch>` for non-default target branches.
- `--draft` when the user asks for a draft MR or the work is intentionally incomplete.
- `--remove-source-branch` when the team convention is to delete branches after merge.
- `--squash` when the MR should request squash-on-merge.

## Credentials

Never put tokens in the command, repository, MR description, or skill files. The helper reads credentials from `GITLAB_TOKEN`.

Expected local setup:

```bash
source ~/.config/gitlab-mr/env
```

If `GITLAB_TOKEN` is missing, tell the user to create or source `~/.config/gitlab-mr/env`. Do not ask the user to paste the token into chat.

## Failure Handling

- If the helper reports no commits ahead of target, do not create an MR; explain that the branch has nothing to publish.
- If an existing MR is returned, report that URL and do not create a duplicate.
- If the user asks for a GitHub PR, do not use this skill; use the GitHub-specific workflow instead.
- If the bundled helper is missing, report that the skill installation is incomplete and reinstall with `npx skills add https://github.com/microHoffman/agent-skills --skill gitlab-create-mr`.

## Expected Output

When successful, report only the essential result:

```text
Created merge request: <url>
```

or:

```text
Existing merge request: <url>
```

If blocked, report the blocker and the exact command or state needed to fix it.
