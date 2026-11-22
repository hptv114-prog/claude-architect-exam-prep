# CI Notes: Claude as a CI Worker

Study notes on using Claude Code in CI/CD. Sections follow the planned outline.

## 1. Get the GitHub CLI (`gh`)

`gh` is the command-line client for GitHub. It lets you work with issues, PRs and Actions runs from the terminal. Claude Code uses it as a tool.

```bash
# macOS
brew install gh
# Debian/Ubuntu
sudo apt install gh
# Windows
winget install --id GitHub.cli

gh auth login     # authenticate (browser or token)
gh auth status    # verify
```

Useful commands:

| Task | Command |
|------|---------|
| List issues | `gh issue list` |
| View an issue | `gh issue view <n>` |
| Create a PR | `gh pr create --title "..." --body "..."` |
| Check PR status | `gh pr checks` |
| List workflow runs | `gh run list` |
| View failed logs | `gh run view <id> --log-failed` |
| Raw API call | `gh api repos/{owner}/{repo}/issues` |

In Actions, `gh` is preinstalled and reads `GH_TOKEN` (usually `${{ secrets.GITHUB_TOKEN }}`).

## 2. CI concepts using a GitHub project

CI (continuous integration) means every change is automatically built and checked before it merges.

Core concepts:

- **Workflow**: a YAML file in `.github/workflows/`.
- **Trigger (`on:`)**: the event that starts it, such as `push`, `pull_request`, `issue_comment` or `schedule`.
- **Job**: a set of steps running on one runner (`runs-on: ubuntu-latest`).
- **Step**: a shell command or a reusable action (`uses:`).
- **Secrets**: encrypted values, referenced as `${{ secrets.NAME }}`.
- **Permissions**: the scope of the `GITHUB_TOKEN`, set with `permissions:`. Use least privilege.
- **Status checks**: pass/fail results shown on a PR. Branch protection can require them.

Example from this repo, `.github/workflows/markdown-only.yml`:

- It triggers on every `push` and `pull_request`.
- It checks out the repo and runs `git ls-files`.
- It fails the job if any non-`.md` file is tracked.

That is CI in miniature: an automated, repeatable gate on every change.

## 3. Live demo: Claude as a CI worker

This repo already has Claude wired up in `.github/workflows/claude.yml`.

Demo flow:

1. Open an issue describing a task.
2. Comment `@claude work on this issue and create a pr`.
3. The `issue_comment` trigger fires the workflow.
4. Claude checks out the repo, creates a branch (`claude/issue-<n>-...`), edits files, commits and pushes.
5. Claude posts progress in a single comment and gives a link to open the PR.
6. The `markdown-only` check runs on the PR.
7. A human reviews and merges.

What to point out:

- Claude works in an ephemeral runner, not on your machine.
- The progress checklist in the comment is the audit trail.
- The human stays in the loop through the PR review.
- Claude can't modify `.github/workflows/` (GitHub App permission limit).

## 4. In-depth configuration of Claude Code as a CI worker

### Action setup

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}  # or anthropic_api_key
    prompt: "..."          # optional; default is the @claude comment text
    claude_args: "--max-turns 10 --allowed-tools Bash(gh pr *)"
    additional_permissions: |
      actions: read
```

### Authentication

- Store `CLAUDE_CODE_OAUTH_TOKEN` or `ANTHROPIC_API_KEY` as a repository secret. Never commit it.
- The workflow needs `id-token: write` for the action's own auth.

### Triggers

| Trigger | Use |
|---------|-----|
| `issue_comment` with `@claude` | Interactive requests on issues and PRs |
| `pull_request` (opened, synchronize) | Automatic code review |
| `issues` (opened, assigned) | Auto-triage or start work |
| `schedule` | Recurring jobs such as dependency or doc audits |

Guard with an `if:` condition (for example `contains(github.event.comment.body, '@claude')`) so the job doesn't run on every event.

### Permissions (least privilege)

- `contents: read` or `write`: write is needed to push branches.
- `pull-requests` and `issues`: read, or write to comment.
- `actions: read`: lets Claude read CI results.
- Grant only what the task needs.

### Tool control

- `--allowed-tools` pre-approves tools. For example, `Bash(gh pr *)` allows only `gh pr` commands.
- `--disallowed-tools` blocks tools.
- Prefer narrow, pattern-scoped Bash permissions over blanket Bash.

### Behavior and cost controls

- `--max-turns` caps the agent loop.
- `--model` selects the model.
- Set a job `timeout-minutes`.
- Use `concurrency:` to avoid overlapping runs on the same issue or PR.

### Guidance via `CLAUDE.md`

A `CLAUDE.md` at the repo root gives Claude standing instructions: conventions, build and test commands, and constraints. This repo's rule is "Markdown files only".

### Safety checklist

- Treat issue and PR text as untrusted input (prompt injection).
- Restrict who can trigger the workflow. The action checks write access by default.
- Keep secrets out of logs and prompts.
- Require human review before merge, with branch protection and required checks.
- Use CI gates (like `markdown-only`) to enforce rules automatically.

## Exam-style takeaways

- CI = automated checks on every change, with triggers defined in workflow YAML.
- `gh` is the scripting interface to GitHub for both humans and agents.
- Claude in CI runs in an ephemeral runner, with scoped permissions and tools.
- Control it through the `prompt`, `claude_args`, `CLAUDE.md` and token permissions.
- Humans review and merge. Required checks enforce policy.
