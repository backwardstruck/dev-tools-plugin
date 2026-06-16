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

```bash
curl -sSL https://raw.githubusercontent.com/backwardstruck/dev-tools-plugin/main/install.sh | bash
```

Then in Claude Code:

```
/plugin marketplace add https://github.com/backwardstruck/dev-tools-plugin
/plugin install dev-tools@dev-tools
```

## Configuration

Set your GitHub org before running the orchestrator:

```bash
export GH_ORG=your-org-name
export GITHUB_TOKEN=ghp_aB3dEfGhIjKlMnOpQrStUvWxYz012345abcd
```

The orchestrator reads both from the environment at startup. You can also hardcode them in your shell profile for convenience:

```bash
# ~/.zshrc
export GH_ORG=backwardstruck
export GITHUB_TOKEN=ghp_aB3dEfGhIjKlMnOpQrStUvWxYz012345abcd
```

## Running the orchestrator

```bash
claude --agent pr-review-orchestrator
```

Or trigger it on a schedule — see `/schedule` in Claude Code.

## Update workflow

After editing any agent:

```bash
git add -A && git commit -m "update" && git push
claude plugin install dev-tools@dev-tools
```

## Permissions

The orchestrator needs `repo` + `read:org` scopes on your GitHub token. Easiest way to set this up:

1. Go to https://github.com/settings/tokens
2. Generate a classic token with full `repo` scope
3. Paste it into `~/.zshrc` as `GITHUB_TOKEN` (see above)

## Troubleshooting

If the orchestrator isn't posting comments, check that `GH_ORG` is set and that your token hasn't expired. The state file lives at `~/.claude/pr-watcher-state.json` — delete it to force a full re-review on next run.

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
- [ ] Move GITHUB_TOKEN to keychain instead of env var (lol)
