# agent-skills

Personal AI agent skills.

This repository is a multi-skill collection installed with skills.sh. It keeps
personal engineering skills that can be installed into Codex or another
supported agent on any machine.

## Skills

### activecollab

Reads and manages ActiveCollab tasks as part of a coding workflow. It gathers
task context, comments, history, subtasks, and attachments through the
[`activecollab` CLI](https://github.com/microHoffman/activecollab-cli), then
uses explicit authorization and CLI dry runs for mutations.

Install only this skill:

```bash
npx skills add https://github.com/microHoffman/agent-skills --skill activecollab
```

This installs the skill only. Install version 0.3.0 or newer of the
[`activecollab` CLI](https://github.com/microHoffman/activecollab-cli/blob/main/docs/installation.md)
separately before using it.

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

Creates, updates, queries, and manages GitHub issues using authenticated `gh`
CLI, REST, and GraphQL operations, including outside-sandbox handling for
permission-sensitive operations.

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

The skill prefers an authenticated `glab` CLI. It also includes its own runnable
fallback helper at:

```text
skills/engineering/gitlab-create-mr/scripts/gitlab-mr
```

No global `gitlab-mr` command is required. Agents should resolve the fallback
helper relative to the installed `SKILL.md`, not relative to the target project
repo.

## Requirements

For `activecollab`:

- [`activecollab` CLI](https://github.com/microHoffman/activecollab-cli)
- CLI version 0.3.0 or newer
- an ActiveCollab account
- a saved CLI login or `ACTIVECOLLAB_URL` and `ACTIVECOLLAB_TOKEN` environment
  variables

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
- preferred: authenticated GitLab CLI, `glab`
- fallback: GitLab access token with permission to push branches and create merge requests

## Credentials

The ActiveCollab skill never asks for or handles credentials. For a self-hosted
server, run the CLI login yourself and pass the complete API-v1 URL:

```bash
activecollab auth login \
  --url https://activecollab.example.com/api/v1
```

The CLI stores the resulting URL, account, and token in its protected per-user
credentials file. To supply an existing token, pipe it from a secret manager
with `--token-stdin`. For CI or ephemeral sessions, the CLI also reads
`ACTIVECOLLAB_URL` and `ACTIVECOLLAB_TOKEN` from the environment. Do not commit
credentials, pass them as command-line arguments, or paste them into an agent
conversation.

The GitLab skill first uses credentials stored by `glab auth login`. If `glab`
is unavailable or unauthenticated for `gitlab.tomatom.cz`, the fallback helper
reads `GITLAB_TOKEN` from the environment.

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

After installing `activecollab`, give the agent a task URL and ask it to inspect
or implement the task. It will read the task context but will not comment,
update fields, create subtasks, or complete work without explicit authorization.

For `gitlab-create-mr`, the skill will:

- check the repository state
- prefer an authenticated `glab` CLI and reuse an existing open MR
- otherwise run the bundled helper's dry run first
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
