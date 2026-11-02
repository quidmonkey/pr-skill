# pr-skill

A Claude Code skill that gets a branch ready for review and opens the pull request.

Running `/pr` on a feature branch goes through six stages in order. Each one has to pass before the next starts:

1. Preflight: checks the git state, gh auth and the base branch, then creates a worktree for the remaining stages.
2. Code review: runs `/code-review --fix` and fixes high and medium findings the branch introduced.
3. Doc review: fixes docs that no longer match the branch.
4. Security review: runs `/security-review` and fixes high and medium findings.
5. Decision records: writes the branch's design decisions as empty commits with `Decision:`, `Rejected:` and `Files:` trailers.
6. Push and PR: pushes the branch and opens or updates the PR, using the repo's PR template if there is one.

Stages 2 to 6 run in a git worktree under `.claude/worktrees/`, so you can keep editing or switch branches while `/pr` works. The worktree only sees commits, so uncommitted changes are left out. When the run ends, the skill fast-forwards your branch to its commits and removes the worktree. If your branch moved in the meantime, it keeps the worktree and tells you which commit to merge.

If a finding can't be fixed, it stops without pushing and reports what's left. Issues that were already on the base branch and touch the branch's code get asked about. Unrelated ones are ignored. Every git hook runs, and the skill never force-pushes.

See [SKILL.md](SKILL.md) for the full behavior.

## Install

```sh
git clone https://github.com/quidmonkey/pr-skill.git ~/.claude/skills/pr
```

You need the [gh CLI](https://cli.github.com/), logged in with `gh auth login`.

## Usage

```
/pr [base] [effort] [--draft]
```

- `base` is the branch the PR targets. It defaults to the repo's default branch.
- `effort` is the review effort passed to `/code-review`: `low`, `medium`, `high` or `xhigh`. The default is `high`.
- `--draft` opens the PR as a draft.

## On the importance of documentation and intent

An agent working on a codebase sees the code but not why it was written that way. Without that context, it "fixes" choices that were made on purpose and works from docs that no longer match the code. The doc review and the decision records exist to give the next agent, or the next reviewer, that context.

The doc review keeps the docs accurate, so an agent reading them gets the current behavior. The decision records keep the intent behind the code where an agent will find it.

### Decision records (ADRs)

Each design decision on the branch becomes an empty commit tagged `[ADR]`, with trailers that say what was chosen, what was rejected and which files it covers:

```
Use Redis for the session store [ADR]

- Sessions move from the in-process dict to Redis

Decision: Redis for sessions so all instances share state
Rejected: sticky sessions, they break on autoscale and deploys
Files: src/session.py, src/config.py
```

A decision is a choice someone might later question or undo, such as a library, a new dependency, or a limit that looks like a bug. The skill takes the reason from commits, comments or the session. If it can't find one, it asks you. The PR title and body carry the `[ADR]` tag and the trailers, so the record survives a squash merge.

On later runs, the code review looks up the decisions covering the files the branch changed, and skips findings that a decision marks as intended.

## License

MIT. See [LICENSE](LICENSE).
