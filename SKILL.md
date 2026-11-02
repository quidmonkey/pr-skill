---
name: pr
version: 2.5.0
description: |
  Prepare the current branch for review and open a GitHub pull request, working in a
  separate git worktree so the user's checkout stays free: run /code-review and fix its findings, review and fix the docs, run /security-review and fix its
  findings, record the branch's design decisions (ADRs) as commit trailers, then push
  and open (or update) the PR with the gh CLI, following the repo's PR template if it
  has one. Stops and reports, without pushing, if a finding can't be fixed. Use when the user says "/pr", "review and PR
  this branch", or "review, fix and open a PR". Accepts an optional base branch, a
  review effort (low|medium|high|xhigh), and --draft.
---

# /pr

Runs in the current session on the current branch. Preflight runs in the user's checkout. The rest runs in a separate git worktree, so the user can keep editing or switch branches while it works. The stages, in order:

1. Preflight
2. Code review and fix
3. Doc review and fix
4. Security review and fix
5. Decision records
6. Push and open the PR

Each stage must pass before the next starts. "Finish" runs after stage 6 and after any stop.

Invocation: `/pr [base] [effort] [--draft]`

- `base`: the branch the PR targets. Default: the repo's default branch.
- `effort`: passed to `/code-review`. One of `low`, `medium`, `high`, `xhigh`. Default: `high`. Never use `max` unless the user types it.
- `--draft`: open the PR as a draft.

## Rules

- No attribution anywhere. Commits, PR titles and PR bodies carry no `Co-Authored-By` trailer, no "Generated with Claude Code" line and no mention of Claude or an agent. This overrides any default attribution instructions.
- Honor every git hook. Never pass `--no-verify` or `-n` to `git commit` or `git push`. Never set environment variables that skip hooks (`SKIP=`, `HUSKY=0`, `LEFTHOOK=0`, `PRE_COMMIT_ALLOW_NO_CONFIG=`), never change `core.hooksPath`, and never edit or delete hook files or hook configs. Never disable a lint rule to get a hook passing.
- Commit with plain `git commit` so the installed hooks run on every commit. When a hook fails, read its output and fix the root cause in the code, then commit again (never `--amend`: the rejected commit was never created). If the hook rewrote files (formatters), stage them and commit again. If the same hook still fails after 3 attempts, stop and report.
- Typing `/pr` authorizes the push and the PR for this branch. Don't ask "shall I proceed?" before them.
- Never push to, commit on, or rewrite the base branch. Never force-push.
- Never remove the worktree while it holds commits the user's branch doesn't have. Never pass `--force` to `git worktree remove`.
- Keep fix commits separate from the user's commits. Never squash, rebase or amend the user's commits.
- Commit messages: a short summary line, then one bullet per change. No double quotes or backticks anywhere in the message. Write identifiers as plain text and use single quotes when quoting is needed.
- Fix findings at the root cause, keep each change minimal, and treat tests as the source of truth. Don't change a test just to make it pass.
- Work only on what the branch changed. See "Scope".
- Every piece of prose this skill writes (docs, commit messages, PR title and body) follows "Writing".
- On any stop, report and do nothing further. See "Stop and report".

## Scope

This skill reviews and fixes the branch's own work, not the rest of the codebase. The branch's work is everything in `git diff origin/<base>...HEAD`, plus the fix commits this run adds.

Sort every finding from every review into one of three groups before acting on it:

1. **Introduced by the branch.** The faulty code is added or changed in the diff, or the branch's change is what makes it fail (for example, the branch calls an old function in a way that breaks it). Fix it. If fixing it needs a decision only the user can make (an intended behavior, an API choice the branch made on purpose), treat it as group 2 instead.
2. **Pre-existing, touching the branch's work.** The faulty code was already on the base branch, but it's in a function the branch edits, or new branch code calls it or relies on it. Don't fix it on your own. Call it out and ask the user (see "Asking the user").
3. **Pre-existing, unrelated to the branch.** Everything else. Don't change it, don't ask about it and don't put it in the PR. Leave it out of the session report too.

When a finding is between groups 2 and 3, put it in group 2.

### Severity

Act only on high and medium severity findings, as a human reviewer would. Skip low severity findings in every group: don't fix them, don't ask about them and don't put them in the PR. If a review already fixed a low severity group 1 finding and the fix is small and passes the checks, keep it; otherwise undo it.

- **High:** a bug or vulnerability that ordinary use or a realistic attacker will hit. Data loss, a crash on normal input, injection, auth bypass, secrets exposed.
- **Medium:** a real bug in a plausible case, a broken documented behavior, a missing check at a trust boundary, or a test that hides a bug.
- **Low:** edge cases that need an unusual setup (a file planted by someone who already has write access, a race in a local tool, an unsupported platform), defense in depth, style, naming, duplication and micro-optimizations.

When a review gives no severity or you disagree with it, judge it yourself using these definitions. When a finding is between two levels, pick the higher one.

### Asking the user

Collect every high or medium group 2 finding from a stage and ask once, at the end of that stage's round, before committing. Use the AskUserQuestion tool, one question per finding (batch up to 4 per call). Each question names the file, line and the problem in one sentence, and offers:

- **Fix it on this branch:** fix it now, in this stage's fix commit.
- **Leave it, note it in the PR:** don't change code. List it under "Notes for reviewers" in the PR body.
- **Leave it:** don't change code and don't mention it in the PR.
- **Stop:** stop the run and report (see "Stop and report").

Follow each answer. A group 2 finding the user has answered doesn't count as open, so it doesn't stop the run or come back in the next round. If a later review reports it again, apply the same answer.

### Briefing the review skills

When invoking `code-review` or `security-review`, add to the args: the worktree path, the range to review (`origin/<base>...HEAD`), and "Fix only high and medium severity issues the branch introduced. Report pre-existing issues without fixing them. For each finding, give its severity (high, medium or low) and say whether it's pre-existing. Don't edit docs (README, docs/, CHANGELOG); a later stage handles them." Then revert any doc edit or out-of-scope fix the review made anyway (`git restore` on those hunks only) before sorting the findings.

## Writing

Write like an engineer leaving a note for a teammate: short, plain and specific.

- Say what changed and why in concrete terms: file, function, behavior, number. Skip anything a reviewer can't act on.
- Short sentences, one idea each. Plain verbs: "is", "has", "adds", "fixes", "removes".
- No filler or hedging: "in order to" is "to", "due to the fact that" is "because", no "could potentially".
- Banned words: additionally, delve, leverage, crucial, pivotal, vital, enhance, robust, seamless, seamlessly, comprehensive, intricate, landscape, tapestry, testament, underscore, showcase, foster, streamline, elevate, empower, boasts, vibrant, rich, cutting-edge, game-changer, holistic, synergy, utilize.
- No "serves as" or "stands as" (write "is"), no "not just X, but Y", no lists of three for rhythm, no "-ing" tails like "..., ensuring X" or "..., highlighting Y".
- No emojis, no em dashes, no mechanical bold, no Title Case headings, no curly quotes, no closing summary or upbeat sign-off.
- Don't describe the review process ("ran code review", "addressed findings") in docs or the PR description. The fix commits already record it.

Before using any text, reread it and cut every word that doesn't carry information.

## 1. Preflight

Run these and stop on the first failure, with the fix it needs:

1. `git rev-parse --show-toplevel` succeeds and `git remote get-url origin` exists.
2. `gh auth status` passes. If it fails, tell the user to run `! gh auth login`.
3. Base: the `base` argument, else `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`. Run `git fetch origin <base>`.
4. The current branch (`git branch --show-current`) is not empty (detached HEAD) and is not `<base>`.
5. `git rev-list --count origin/<base>..HEAD` is greater than 0.
6. `git status --porcelain` is empty. If it isn't, show the changed files and say the worktree only sees commits, so these changes won't be reviewed or pushed. Ask whether to commit them first (as the user's own commit, through the hooks), continue without them, or stop. Don't stash or discard them.
7. Run `gh pr view --json url,state,isDraft,body` to see whether the branch already has a PR. Remember the result for stage 6.
8. Create the worktree. See "Worktree".

## Worktree

Stages 2 to 6 run in a worktree at `<toplevel>/.claude/worktrees/pr-<branch>`, where `<toplevel>` is the output of `git rev-parse --show-toplevel` and `<branch>` is the current branch with each `/` written as `-`.

1. If that path exists, stop and report: an earlier run left it behind, likely holding fix commits. Tell the user to check it and remove it with `git worktree remove <path>`.
2. If `git check-ignore -q .claude/worktrees/x` fails, append `.claude/worktrees/` to `$(git rev-parse --git-common-dir)/info/exclude`, so the worktree doesn't show up as untracked in the user's checkout. Don't edit `.gitignore`.
3. `git worktree add --detach <path> <branch>`. The worktree's HEAD is detached at the branch tip, because git won't check out one branch in two worktrees. Commits made there move only the worktree's HEAD until "Finish" brings them to the branch.
4. Switch the session into it with the EnterWorktree tool, passing `path`. Every later command, review skill and commit runs there. Where a later step says the current branch, it means `<branch>`.

## Running checks

After every round of fixes, before committing (the commit runs the git hooks, so don't run them here):

- Run the project's test command if one is evident (Makefile target, `pyproject.toml`, `package.json` scripts). Run lint too if it's separate from the hooks.
- Fix failures at the root cause and run again until clean.

## 2. Code review and fix

Before the first round, read the recorded decisions and the docs covering the changed code:

```
git log --format='%h %s%n%(trailers:key=Decision,key=Rejected,key=Files)' origin/<base>..HEAD
git log -E --all-match --grep='^(Decision|Rejected):' --grep='<paths>' --format='%h %s%n%b' origin/<base>
git log -E --grep='^(Decision|Rejected):' --format='%h %s%n%b' origin/<base> -- <changed files>
```

The first lists the branch's decisions. The other two find decisions already on the base that cover the branch's changed files (`git diff --name-only origin/<base>...HEAD`). The second matches decision records whose `Files:` line names a changed file; `<paths>` is the changed files joined with `|`, with each `.` written as `\.`. It's needed because a decision commit changes no files, so a path filter can't find it. The third finds decisions in commits that changed those files, such as squash merges. Both match lines instead of parsing trailers because a squash merge can push trailers into the middle of a message. Read the union.

Don't fix a finding that a recorded decision or a doc marks as intentional, unless it causes a concrete bug. If the branch goes against a decision recorded on the base, note it for stage 5.

Up to 2 rounds:

1. Invoke the `code-review` skill with args `<effort> --fix` plus the brief from "Briefing the review skills".
2. If it reports no high or medium findings, this stage passes (after handling any low severity fixes it made, per "Severity").
3. Sort the findings (see "Scope"). Review what `--fix` changed and undo anything outside group 1. Fix any group 1 finding it left unresolved. Skip any finding that a trailer or doc rules out, and note it for the PR body. Ask about group 2 findings.
4. If nothing changed, this stage passes. Otherwise run "Running checks", then commit:

   ```
   Address code review findings

   - <one bullet per fix>
   ```

5. Start the next round.

If high or medium group 1 findings remain after round 2, stop and report.

## 3. Doc review and fix

Check that the repo's docs match the branch as it is now, including the code review fixes.

1. Find the docs: `README.md`, `ARCHITECTURE.md`, `CONTRIBUTING.md`, everything under `docs/`, a `CHANGELOG.md`, and READMEs in the directories the branch touched. Skip generated docs, vendored code, and plan or implementation docs written for agents.
2. Read the full branch diff: `git diff origin/<base>...HEAD`.
3. Look for drift. A doc needs a change when the branch:
   - changes design, architecture, or the boundaries between modules
   - adds, removes or renames a public function, CLI command, flag, config key, env var, endpoint or file path that a doc mentions or should mention
   - changes documented behavior, defaults, setup steps or examples
   - removes or renames anything a doc still names (grep the docs for each removed or renamed identifier)
4. Fix each drift with the smallest edit that makes the doc accurate. Match the doc's existing structure and tone. Follow "Writing".
   - Don't create new doc files, and don't add docs for internal changes nobody reads about.
   - Update `CHANGELOG.md` only if it keeps an unreleased section; add one line in its existing format.
   - Don't edit code comments or docstrings here. That's the code review's job.
   - Fix only drift the branch caused. Leave doc errors that were already on the base branch alone, unless they describe code the branch changed; then ask, as in "Asking the user".
5. If nothing drifted, this stage passes with no commit.
6. Otherwise run "Running checks" on the changed docs, then commit:

   ```
   Update docs for branch changes

   - <one bullet per doc changed>
   ```

7. Reread each edited section against the diff once. Fix anything still wrong in a second commit. If a doc can't be made accurate without a decision from the user (for example, which of two behaviors is intended), stop and report.

## 4. Security review and fix

Up to 2 rounds:

1. Invoke the `security-review` skill with the brief from "Briefing the review skills". It reports only. It doesn't edit code.
2. If it reports no high or medium findings, this stage passes.
3. Sort the findings (see "Scope"). Fix each group 1 finding at its root cause. If a fix changes documented behavior, update that doc in the same commit, following "Writing". If you're confident a finding is a false positive, don't change code. Record the finding and the reason for the PR body. Ask about group 2 findings.
4. If nothing changed, this stage passes. Otherwise run "Running checks", then commit:

   ```
   Address security review findings

   - <one bullet per fix>
   ```

5. Start the next round.

If a real high or medium group 1 finding remains after round 2, stop and report.

If this stage changed code, run the `code-review` skill once more with `<effort> --fix` and the brief, on the new commits. Handle any findings as in stage 2, within one extra round. Stop and report if they remain.

## 5. Decision records

Record the branch's design decisions in git history, where `git log` finds them later. Each decision gets its own commit. The subject states the decision, so `git log --oneline` and the recent-commit list an agent sees at session start carry it without opening the commit. The trailers hold the reason, the rejected alternative and the files the decision covers.

1. Collect the decisions already recorded: the branch's trailers and the base decisions from stage 2.
2. Read `git diff origin/<base>...HEAD` and list the decisions it makes that have no record. A decision is a choice a later reviewer or agent might question or undo:
   - an approach picked over an obvious alternative (a library, a data model, an algorithm, an API shape)
   - a new dependency, service or data store
   - an intended limit, omission or behavior that looks like a bug
   - a reversal of a decision recorded on the base

   Renames, refactors with no behavior change, bug fixes with one obvious fix, and tests are not decisions. Most branches have none, one or two.
3. Find each decision's reason in the branch's commit messages, code comments, the existing PR body or this session's conversation. Never invent a reason. When none is stated, ask the user with AskUserQuestion, one question per decision (batch up to 4 per call). Each question names the change and the files in one sentence and offers:
   - **Record it:** record the drafted Decision and Rejected lines, shown in the option description. The user can write their own through Other.
   - **Not a decision:** don't record it.
4. A decision that reverses one on the base names the old commit: `Decision: <what and why>. Replaces the decision in <short sha>.`
5. If nothing is missing, this stage passes with no commit. Otherwise make one commit per decision with `git commit --allow-empty`, through the hooks. The commits are empty because this skill never amends the user's commits.

   ```
   Use Redis for the session store [ADR]

   - Sessions move from the in-process dict to Redis

   Decision: Redis for sessions so all instances share state
   Rejected: sticky sessions, they break on autoscale and deploys
   Files: src/session.py, src/config.py
   ```

   - Subject: the decision as an imperative sentence under 72 characters, ending in `[ADR]`. If the base's recent subjects (`git log --format=%s -n 20 origin/<base>`) follow conventional commits, write `docs(adr): <decision>` instead, so commit-msg hooks and changelog tools accept it.
   - Body: one bullet saying what changed.
   - Trailers: `Decision:` with what was chosen and why, `Rejected:` with the alternative and why not (leave it out if none was considered), and `Files:` with the repo-relative paths the decision covers, comma-separated. Stage 2 finds decisions by these paths, so list every file the decision governs, not only the ones this branch changed. Keep the trailers in the last paragraph with no blank lines between them, so git parses them.
   - Follow "Writing".

## 6. Push and open the PR

1. `git push origin HEAD:refs/heads/<branch>`. This runs the installed `pre-push` hook. If it fails, fix the root cause, commit through the hooks, and push again (up to 3 attempts, then stop and report).
2. Write the title. See "Title".
3. Write the body to a file in the scratchpad directory. See "Body".
4. No PR yet: `gh pr create --base <base> --head <current branch> --title '<title>' --body-file <file> [--draft]`.
   An open PR already: `gh pr edit <branch> --title '<title>' --body-file <file>`. Keep any text a person wrote in its existing body; rewrite only the parts this skill wrote. Don't change its draft state or base.
   A closed or merged PR: stop and report. Don't reopen it or create a second one.

Run "Finish", then report the PR URL and the commits this run added.

### Title

A plain sentence a teammate understands without opening the PR. Imperative mood, sentence case, no trailing period, under 70 characters. It says what the PR does for the codebase, for example "Retry failed uploads with backoff", not the branch name, a ticket number alone, or "Updates".

If the branch records any decisions (its own trailers or the stage 5 commits), end the title with ` [ADR]`. A squash merge uses the PR title as the commit subject, so the marker reaches the base.

Match the repo's title convention. Look at `gh pr list --state merged --limit 10 --json title`. If those titles use a prefix (for example `feat:`, `[ABC-123]`), use the same form and take the ticket ID from the branch name if it has one.

### Body

Find a PR template, case-insensitive, in this order: `.github/pull_request_template.md`, `pull_request_template.md` at the root, `docs/pull_request_template.md`, then the `.github/PULL_REQUEST_TEMPLATE/`, `PULL_REQUEST_TEMPLATE/` or `docs/PULL_REQUEST_TEMPLATE/` directory. If a directory holds several templates, pick the one whose name fits the change (for example `bugfix.md` for a fix). Ask the user if none fits clearly.

With a template:

- Keep its headings, order and checklists exactly as written.
- Fill each section that asks for something with short, specific text, following "Writing". Write only what the section asks for.
- Remove instructional HTML comments (`<!-- ... -->`) once their section is filled. Leave other comments in place.
- Check a checkbox only when this run verified it (for example, tests pass, docs updated). Leave the rest unchecked. Never check a box for something a human has to confirm.
- Write "N/A" in a section that doesn't apply. Don't delete it.
- Put disputed review findings and false positives in the section for reviewer notes. If the template has none, add a short "Notes for reviewers" section at the end.

Without a template:

```
<One to three sentences: what the PR changes and why.>

- <one bullet per meaningful change, from the commit subjects; leave out the review and doc fix commits unless they changed behavior>

Notes for reviewers
- <only if needed: a disputed finding or false positive and the reason, a pre-existing issue the user chose to note, a decision the reviewer should check>
```

Leave out "Notes for reviewers" when it would be empty. No other sections.

If the branch records any decisions, end the body with their `Decision:`, `Rejected:` and `Files:` lines as the last paragraph, with no heading, after any template sections and notes. A squash merge can use the PR body as the commit message, and git reads trailers only from the last paragraph. When updating a PR, replace that paragraph instead of adding a second one.

Before creating the PR, reread the title and body against "Writing" and cut anything that doesn't help a reviewer.

## Finish

Bring the worktree's commits to the user's branch and remove the worktree. Run this after stage 6 and after any stop once this run has created the worktree.

1. Note the worktree's HEAD sha (`git rev-parse HEAD`), then leave with the ExitWorktree tool, `action: "keep"`. The session is back in the user's checkout.
2. If that sha differs from `<branch>`, bring the branch up to it:
   - If the user's checkout is on `<branch>`, run `git merge --ff-only <sha>`. Git refuses if the user committed to the branch meanwhile or has uncommitted edits to the same files. It leaves their other edits alone.
   - If the user switched to another branch, run `git branch -f <branch> <sha>`, but only when `git merge-base --is-ancestor <branch> <sha>` passes.
3. If stage 6 pushed and `<branch>` has no upstream, run `git branch --set-upstream-to=origin/<branch> <branch>`.
4. If step 2 succeeded or had nothing to do, run `git worktree remove <path>`. If step 2 failed, keep the worktree and report its path, the sha, and what blocks the fast-forward (new commits on the branch, or uncommitted edits that collide). Tell the user to merge or rebase onto that sha, then remove the worktree.

## Stop and report

When a stage can't pass, don't push and don't create or edit a PR. Leave every commit made so far in place and don't undo anything. Run "Finish" if this run created the worktree. Report:

- the stage that stopped (preflight, code review, doc review, security review, decision records, push)
- each unresolved high or medium group 1 finding or failing hook, with file and line and the output that matters
- what was fixed and committed before the stop
- whether "Finish" brought the commits to the branch, or the worktree path it kept
- what the user needs to do, then that running `/pr` again picks up from the current branch state
