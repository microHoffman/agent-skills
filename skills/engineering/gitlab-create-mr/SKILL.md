---
name: gitlab-create-mr
description: Create GitLab merge requests for repositories hosted on gitlab.tomatom.cz, preferring an authenticated glab CLI and falling back to the bundled gitlab-mr helper with GITLAB_TOKEN. Use this skill whenever the user asks Codex to create, open, publish, submit, or prepare a GitLab MR/merge request from a local branch after code changes are committed. This is for self-hosted GitLab merge requests, not GitHub pull requests or code-review-only tasks.
---

# GitLab Create MR

## Overview

Prefer the authenticated `glab` CLI to push the current branch and create a
GitLab merge request. If `glab` is unavailable or not authenticated for
`gitlab.tomatom.cz`, use this skill's bundled `scripts/gitlab-mr` helper.

Resolve the helper path relative to this installed skill directory, not relative
to the user's project repository. For example, if this file is installed at
`<skill-dir>/SKILL.md`, run `<skill-dir>/scripts/gitlab-mr`.

## Workflow

1. Confirm the repo state before creating an MR:
   - run `git status --short --branch`
   - ensure the intended changes are already committed; the helper publishes committed branch state only
   - local uncommitted or untracked files are not included in the MR
2. Inspect the committed change before writing MR metadata:
   - identify the target branch first, then use `git log --oneline <target>..HEAD`, `git diff --stat <target>..HEAD`, and focused diffs as needed
   - infer the user-facing purpose of the branch from the actual code changes, not just the branch name or latest commit subject
3. Decide whether the user provided explicit MR metadata:
   - if the user's prompt clearly specifies a title/name, use that exact title instead of generating one
   - if the user's prompt clearly specifies a description/body, use that exact description instead of generating one
   - if only one field is explicit, generate only the missing field from the inspected change
   - do not treat vague requests like "create a PR" or branch names as explicit metadata
4. Build AI-written MR metadata for any field the user did not explicitly provide:
   - title: short imperative or noun phrase describing the real change, not a placeholder like "test commit"
   - description: 1-3 concise sentences, or a short `Summary` / `Verification` format when useful
   - mention verification results only when tests/checks were actually run, and summarize outcomes instead of listing raw commands by default
   - never include shell transcripts, literal `\n` escape text, credentials, or unrelated local status in the MR description
   - if the diff is too small or unclear to infer meaningful intent, use a factual description of the exact change
5. Select the publishing path:
   - if `glab` is installed, run `glab auth status --hostname gitlab.tomatom.cz`
   - if that check fails only because sandboxed network or DNS access is blocked, retry it with scoped network escalation before treating `glab` as unauthenticated
   - never use `--show-token` and never copy the token from `glab` configuration into another credential store
6. When `glab` is installed and authenticated:
   - check `glab mr list --source-branch "<source>" --target-branch "<target>" --output json` and report an existing open MR instead of creating a duplicate
   - push with `git push -u origin <source>`
   - create the MR non-interactively with `glab mr create --source-branch "<source>" --target-branch "<target>" --title "<title>" --description "<summary and test notes>" --yes`
7. Otherwise, use the bundled helper:
   - ensure `GITLAB_TOKEN` is available; if it is not already exported and `~/.config/gitlab-mr/env` exists, source that file in the helper's shell invocation
   - run `<skill-dir>/scripts/gitlab-mr --dry-run --title "<title>" --description "<summary and test notes>"`
   - if the dry run is correct, run the same helper without `--dry-run`
8. For multiline descriptions, pass real newline characters, not literal `\n` text.
9. Report the returned GitLab MR URL.

## Command Options

Use these only when they match the user's request or repo conventions:

- Target branch: `glab --target-branch <branch>` or helper `--target <branch>`.
- Draft: `--draft` with either path.
- Remove source branch: `--remove-source-branch` with either path.
- Squash: `glab --squash-before-merge` or helper `--squash`.

## Credentials

Never put tokens in a command, repository, MR description, or skill file.
Prefer credentials already stored by `glab auth login`. Do not display, extract,
or duplicate the stored token.

The fallback helper reads `GITLAB_TOKEN` and checks it during dry-run and real
runs, so missing credentials are caught before the branch push and GitLab API
request.

Expected local setup:

```bash
source ~/.config/gitlab-mr/env
```

If both `glab` authentication and `GITLAB_TOKEN` are unavailable, tell the user
to authenticate `glab` or create/source `~/.config/gitlab-mr/env`. Do not ask
the user to paste a token into chat.

## Failure Handling

- If the helper reports no commits ahead of target, do not create an MR; explain that the branch has nothing to publish.
- If `glab` is missing or genuinely unauthenticated, continue to the helper instead of stopping.
- If a `glab` authentication check fails because of sandboxed network or DNS access, retry with scoped network escalation before falling back.
- If the helper reports that `GITLAB_TOKEN` is missing during dry-run, source `~/.config/gitlab-mr/env` in the same shell invocation when that file exists and repeat the dry-run.
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
