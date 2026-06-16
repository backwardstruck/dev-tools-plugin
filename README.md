# dev-tools-plugin

Claude Code plugin for automated PR review. Runs a multi-agent loop across your GitHub org — discovers open PRs, dispatches specialist sub-agents (frontend, backend, infra, security), aggregates their findings, and posts a consolidated review comment on each PR.

## Agents

| Agent | What it does |
|---|---|
| `pr-review-orchestrator` | Main loop — discovers, routes, aggregates, posts |
| `frontend-reviewer` | React, TypeScript, CSS/Tailwind |
| `backend-reviewer` | Python, FastAPI, SQLAlchemy, migrations |
| `infra-reviewer` | CDK, Dockerfiles, GitHub Actions, Nginx |
| `security-reviewer` | Runs on every PR regardless of content area |
| `pr-escalation-manager` | Personal EM triage — assigned/watching/team modes |

## Quick install

In Claude Code:

```
/plugin marketplace add https://github.com/backwardstruck/dev-tools-plugin
/plugin install dev-tools@dev-tools
```

## Configuration

Set your GitHub org before running the orchestrator:

```bash
export GH_ORG=your-org-name
```

**Do not set `GITHUB_TOKEN` as an environment variable or in your shell profile.** Instead, authenticate via `gh`:

```bash
gh auth login
```

`gh auth login` stores your token in the system keychain — not in plaintext on disk. The orchestrator uses the `gh` CLI for all GitHub operations, so a valid `gh` session is all it needs.

## Running the orchestrator

```bash
claude --agent pr-review-orchestrator
```

Or trigger it on a schedule — see `/schedule` in Claude Code.

## Update workflow

After editing any agent, stage files explicitly and reinstall:

```bash
git add plugins/dev-tools/agents/<changed-file>.md \
        plugins/dev-tools/.claude-plugin/plugin.json \
        .claude-plugin/marketplace.json
git commit -m "describe the change"
git push
claude plugin install dev-tools@dev-tools
```

## Permissions

The orchestrator needs `public_repo` (or `repo` for private repos), `read:org`, and `repo:status` scopes. Use a **fine-grained personal access token** scoped to only the repositories you want reviewed — not a classic token with full `repo` access.

1. Go to https://github.com/settings/tokens?type=beta
2. Create a fine-grained token scoped to your target repos
3. Grant: **Contents** (read), **Pull requests** (read/write), **Metadata** (read)
4. Authenticate: `gh auth login --with-token <<< your-token`

## Troubleshooting

If the orchestrator isn't posting comments, check that `GH_ORG` is set and that your `gh` session is active (`gh auth status`). The state file lives at `~/.claude/pr-watcher-state.json` — delete it to force a full re-review on next run. Do not commit or share the state file — it contains org names and PR metadata.

If you hit rate limits, reduce `PR_WATCHER_MAX_PRS` (default 30).

## Architecture

```
pr-review-orchestrator
├── discovers repos (gh repo list)
├── discovers PRs (gh pr list)
├── fetches diffs (gh pr diff)
├── dispatches specialists in parallel
│   ├── frontend-reviewer
│   ├── backend-reviewer
│   ├── infra-reviewer
│   └── security-reviewer (always)
└── aggregates + posts consolidated comment
```

State is persisted to `~/.claude/pr-watcher-state.json` between runs so already-reviewed SHAs aren't re-reviewed.

## TODO

- [ ] Add webhook support so reviews trigger on push instead of polling
- [ ] Wire up Slack notifications for bucket 2 escalations
- [ ] Migrate token management to fine-grained PATs scoped per-repo (tracked: permissions section above)
