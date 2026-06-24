# agent-skills

Personal AI agent skills.

This repository is a multi-skill collection installed with skills.sh. It keeps
personal engineering skills that can be installed into Codex or another
supported agent on any machine.

## Skills

### tanstack-query-angular

Builds, refactors, reviews, and debugs TanStack Query usage in Angular
applications using `@tanstack/angular-query-experimental`. Use it for Angular
Query setup, query keys, signal-aware `injectQuery`, mutations, invalidation,
pagination, tests, and migrations from manual `HttpClient`/RxJS server-state
handling.

Install only this skill:

```bash
npx skills add https://github.com/microHoffman/agent-skills --skill tanstack-query-angular
```

### create-pull-request

Creates GitHub pull requests with project conventions, PR templates, detailed
commit-message expectations, and proactive outside-sandbox handling for GitHub
CLI operations.

Install only this skill:

```bash
npx skills add https://github.com/microHoffman/agent-skills --skill create-pull-request
```

### github-issues

Creates, updates, and manages GitHub issues using MCP tools for reads and
authenticated `gh` CLI operations for writes, including outside-sandbox
handling for permission-sensitive operations.

Install only this skill:

```bash
npx skills add https://github.com/microHoffman/agent-skills --skill github-issues
```

### gitlab-create-mr

Creates GitLab merge requests for repositories hosted on `gitlab.tomatom.cz`.
It is intended for workflows where the agent has already implemented and
committed a change, then needs to push the current branch and open a merge
request.

Install only this skill:

```bash
npx skills add https://github.com/microHoffman/agent-skills --skill gitlab-create-mr
```

The skill includes its own runnable helper at:

```text
skills/engineering/gitlab-create-mr/scripts/gitlab-mr
```

No global `gitlab-mr` command is required. Agents should resolve this helper
relative to the installed `SKILL.md`, not relative to the target project repo.

## Requirements

For `tanstack-query-angular`:

- Angular application codebase
- `@tanstack/angular-query-experimental`
- Current TanStack Angular Query docs when exact experimental API syntax matters

For `create-pull-request` and `github-issues`:

- `git`
- GitHub CLI, `gh`
- authenticated GitHub access through `gh auth login`

For `gitlab-create-mr`:

- `bash`
- `git`
- `curl`
- `jq`
- GitLab access token with permission to push branches and create merge requests

## Credentials

The helper does not store credentials. It reads `GITLAB_TOKEN` from the
environment.

Recommended local setup:

```bash
mkdir -p ~/.config/gitlab-mr
chmod 700 ~/.config/gitlab-mr
```

Create `~/.config/gitlab-mr/env`:

```bash
export GITLAB_TOKEN="..."
export GITLAB_HOST="gitlab.tomatom.cz"
export GITLAB_SCHEME="https"
```

Then load it when needed:

```bash
source ~/.config/gitlab-mr/env
```

Do not commit the real `env` file.

## Agent Usage

After installing a skill, ask the agent for the matching GitHub or GitLab
workflow. For `gitlab-create-mr`, the skill will:

- check the repository state
- run a dry run first
- push the branch
- create or reuse an open merge request

The helper publishes committed branch state only. Local uncommitted or
untracked files are not included in the merge request.

The helper fails before pushing or creating an MR when:

- it is not run inside a Git repo
- `origin` is missing or not hosted on `gitlab.tomatom.cz`
- the current branch is the target branch
- the branch has no commits ahead of the target branch
- `GITLAB_TOKEN` is missing for a real run
