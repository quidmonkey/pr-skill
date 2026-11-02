# pr-skill

A Claude Code skill that gets a branch ready for review and opens the pull request.

Running `/pr` on a feature branch goes through six stages in order. Each one has to pass before the next starts:

1. Preflight: checks the git state, gh auth and the base branch.
2. Code review: runs `/code-review --fix` and fixes high and medium findings the branch introduced.
3. Doc review: fixes docs that no longer match the branch.
4. Security review: runs `/security-review` and fixes high and medium findings.
5. Decision records: writes the branch's design decisions as empty commits with `Decision:`, `Rejected:` and `Files:` trailers.
6. Push and PR: pushes the branch and opens or updates the PR, using the repo's PR template if there is one.

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

## License

MIT. See [LICENSE](LICENSE).
