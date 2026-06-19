# AGENTS.md — global template

This is a **project-agnostic** agent-guidelines file. It codifies the
development workflow that has worked well across the `~/pse/*` repos:
TDD, `backlog` for progress tracking, `jj` (Jujutsu) for version control,
and a strict PR review loop built on `roborev` + GitHub reviewer bots.

Each consuming repo replaces this top section with a project-specific
header (project name + one-line description) and appends a
`## Project-specific` section at the bottom (build commands, layout,
special invariants). Everything between is shared.

See `README.md` for installation options (symlink vs copy vs submodule).

## Development Workflow

### TDD (Test-Driven Development)
- Write tests FIRST before implementing features
- Red → Green → Refactor cycle
- Run tests frequently during development

### Progress Tracking
- Use `backlog` CLI to track all work (`backlog task list`, `backlog task edit`, `backlog task view`)
- Update the relevant backlog task status/notes when starting or completing work
- Treat any `docs/KANBAN.md` as historical archive only (do not use it for active tracking)

### Version Control — `jj` (Jujutsu)
Use `jj` instead of `git` for all VCS operations. Common commands:
- `jj status` — show working copy changes
- `jj log` — show commit history
- `jj new -m "message"` — create new change
- `jj describe -m "message"` — update current change description
- `jj squash` — squash into parent
- `jj rebase -s <rev> -d <dest>` — rebase a change (and descendants) onto a destination
- `jj bookmark set <name> -r <rev>` — create/move a bookmark (needed before `jj git push`)
- `jj git push --bookmark <name>` — push (sideways moves are safe force-with-lease)
- `jj git import` / `jj git export` — sync with the underlying git repo (auto in colocated workspaces)
- `jj metaedit --update-author` — fix author info

Notes:
- `change_id` is stable across rewrites; `commit_id` (git SHA) is not. Reference prior work by `change_id` in backlog notes when possible.
- A "mutable" jj change will get a new git SHA every time you amend it; do NOT self-reference SHAs in commit messages or notes — they go stale.
- For PRs: keep one logical change per bookmark. Use `jj new main@origin -m "..."` to start a fresh branch.

### Pull Request review loop (MANDATORY — do not skip)
Every PR must complete the FULL review loop before merge. Roborev is one
input, not the whole loop.

1. **Run roborev 2×2** (`codex` + `claude-code` × `security` + `design`) on
   the pushed SHA. Enqueue sequentially (not in a parallel `&` batch) to
   avoid the daemon race where 3 of 4 jobs fail with
   `repo_path and git_ref are required`:
   ```bash
   roborev review "<SHA>" --repo "$PWD" --agent codex       --model gpt-5.5 --type security
   roborev review "<SHA>" --repo "$PWD" --agent codex       --model gpt-5.5 --type design
   roborev review "<SHA>" --repo "$PWD" --agent claude-code --model opus    --type security
   roborev review "<SHA>" --repo "$PWD" --agent claude-code --model opus    --type design
   ```
2. **Wait for the GitHub bot reviews to land** before triaging — they are
   asynchronous and inline findings can arrive minutes after the push.
   Poll the PR's reviews/comments (see step 3) until each expected bot
   (`gemini-code-assist[bot]`, `coderabbitai[bot]`, `Cursor Bugbot`,
   `greptile-apps[bot]`) has
   either posted or timed out. If a bot never responds, record that
   explicitly ("timed out / not installed") in the PR body so the absence
   is deliberate, not accidental.
3. **Read the GitHub bot reviews**, not just roborev:
   - `gemini-code-assist[bot]` — inline comments (often the most actionable)
   - `coderabbitai[bot]`
   - `Cursor Bugbot`
   - `greptile-apps[bot]` — static analysis that catches cfg-gated
     compile failures the others miss (see lessons below)
   - any other installed reviewer bot
   Inspect with (replace `<OWNER>`/`<REPO>`/`<N>`):
   ```bash
   gh api repos/<OWNER>/<REPO>/pulls/<N>/reviews   --jq '.[] | {user,state,body}'
   gh api repos/<OWNER>/<REPO>/pulls/<N>/comments  --jq '.[] | {user,path,line,body}'
   gh api repos/<OWNER>/<REPO>/issues/<N>/comments --jq '.[].user.login'
   ```
4. **Triage every inline finding** (action / defer-with-reason / reject-with-reason).
   Actionable HIGH/MEDIUM findings MUST be fixed in the CURRENT PR before
   merge — do NOT merge with an actionable HIGH/MEDIUM finding outstanding.
   A follow-up PR is acceptable only for (a) historical findings on already-
   merged code, or (b) findings you explicitly defer with a recorded
   rationale in both the PR body and the review thread reply.
5. **Reply on the thread** for each actionable inline comment
   (`POST .../pulls/<N>/comments/<id>/replies`), linking the follow-up PR if
   the fix lands separately. Replies MUST go on the **same PR that owns
   the comment** — `comment_id` is scoped to its PR:
   ```bash
   gh api -X POST repos/<OWNER>/<REPO>/pulls/<N>/comments/<comment_id>/replies \
     -f body="Fixed in this PR via ..." --jq '{created: (.created_at != null)}'
   ```
6. **Resolve the thread** after replying — GitHub "Resolve conversation"
   is GraphQL-only (no REST endpoint). Get the thread node_id, then:
   ```bash
   gh api graphql -f query='{ repository(owner: "<OWNER>", name: "<REPO>") {
     pullRequest(number: <N>) { reviewThreads(first: 30) { nodes {
       id isResolved comments(first: 1) { nodes { databaseId } } } } } } }' \
     --jq '.data.repository.pullRequest.reviewThreads.nodes[]
           | select(.isResolved == false)
           | {thread_id: .id, comment_id: .comments.nodes[0].databaseId}'
   # Then resolve each:
   gh api graphql -f query='mutation { resolveReviewThread(input:
     {threadId: "PRRT_..."}) { thread { isResolved } } }'
   ```
   Never leave a thread open after triage — even on merged PRs.
7. **Close the roborev job** once findings are addressed or explicitly
   deferred: `roborev close <job_id>` (alias `address`). Open reviews are
   not a backlog — they are state that must be resolved per-PR.
8. **Compact regularly.** After merging a series of PRs (or any time
   `roborev list --open --limit 200` shows > 10 open reviews), run:
   ```bash
   roborev compact --all-branches --wait --limit 50 --timeout 15m
   ```
   This consolidates duplicates, drops false positives / already-fixed
   findings, and auto-closes the originals. The resulting consolidated job
   must itself be triaged (step 4) and closed (step 7). Do this across ALL
   branches (`--all-branches`) — reviews live on whatever feature branch
   the PR used, not just `main`.
9. **If fixes are needed**, open a follow-up PR and re-run steps 1–8 on it.

### Pre-merge checklist (run this BEFORE every `gh pr merge`)

```bash
# 1. CI is CLEAN (all required checks SUCCESS)
gh pr view <N> --json mergeStateStatus --jq '.mergeStateStatus'  # must print CLEAN

# 2. greptile has posted (it is the slowest bot — 5-10 min after push).
#    Check BOTH inline comments AND reviews (greptile may post a top-level
#    review with no inline comment):
gh api repos/<OWNER>/<REPO>/pulls/<N>/comments \
  --jq '[.[] | select(.user.login == "greptile-apps[bot]")] | length'
gh api repos/<OWNER>/<REPO>/pulls/<N>/reviews \
  --jq '[.[] | select(.user.login == "greptile-apps[bot]")] | length'
# If BOTH are 0 AND the PR was pushed <10 min ago: WAIT. Do not merge.
# greptile catches cfg-gated compile failures and AC/notes consistency
# issues that gemini + roborev miss.

# 3. No unresolved review threads (paginate if >50 threads)
gh api graphql -f query='{ repository(owner: "<OWNER>", name: "<REPO>") {
  pullRequest(number: <N>) { reviewThreads(first: 50) {
    nodes { isResolved } pageInfo { hasNextPage } } } } }' \
  --jq 'if .data.repository.pullRequest.reviewThreads.pageInfo.hasNextPage
        then "ERROR: >50 threads, paginate manually"
        else [.data.repository.pullRequest.reviewThreads.nodes[]
              | select(.isResolved == false)] | length
        end'  # must print 0

# 4. roborev jobs closed
roborev list --open --limit 200 --json --repo "$PWD" \
  --branch <BRANCH> | jq 'length'  # must print 0

# 5. PR body's review-loop checkboxes are ALL marked [x] (status report,
#    not a to-do list). A merged PR with unchecked "roborev 2x2" / "threads
#    resolved" / "jobs closed" reads as "loop was never completed" to
#    future readers (and future-you in the next session). Scope the grep to
#    the review-loop section so a legitimately-unchecked box elsewhere in
#    the body (deferred task, future-work list) does not false-block.
gh pr view <N> --json body --jq '.body' | grep -cE '^- \[ \].*(roborev|bot review|threads|jobs closed|triaged)'  # must print 0
# If any review-loop `[ ]` line remains, update the PR body before merging:
#   gh pr edit <N> --body-file <updated-with-[x]>
```

Only after ALL FIVE pass: `gh pr merge <N> --merge --delete-branch`.

### Post-merge branch cleanup (run after every `gh pr merge`)
The remote branch is auto-deleted only if you pass `--delete-branch` AND no
manual force-push re-created it; local branches are NEVER auto-deleted. Stale
branches accumulate silently across a PR series and obscure what is actually
open. Clean up immediately after each merge, not "later":

```bash
# 1. List local + remote branches that still exist.
git fetch --prune origin
git branch -a

# 2. For each non-`main` branch, check its PR state via gh (NOT `git --merged`).
#    In squash-merge repos `git branch --merged main` is USELESS — GitHub mints
#    a fresh SHA on `main`, so the feature branch's tip is never in main's
#    history even after a clean merge. `gh` reads the actual PR state:
gh pr list --state merged --head <branch> --json number,state,mergedAt

# 3. Delete the branch in BOTH places if its PR is MERGED (or it has no PR
#    and is provably dead — e.g. a local scratch branch):
git branch -d <branch>                       # local; refuses (-d exits non-zero)
                                             # for squash-merged branches — see below
git push origin --delete <branch>            # remote — errors (NOT a no-op)
                                             # if already gone; append `|| true`
                                             # under `set -e`

# 4. Re-prune remote-tracking refs that the deletes just orphaned:
git fetch --prune origin
```

Step 3 notes:
* `git branch -d` refuses a squash-merged branch (its tip is not in `main`'s
  history, so `-d`'s safety check fails). After confirming `MERGED` via `gh`
  in step 2, force with `git branch -D <branch>`. In a regular-merge repo `-d`
  works and IS the safer check — keep it where you can.
* `gh pr merge --delete-branch` deletes the REMOTE at merge time, but a LOCAL
  ref always remains, and a re-push (or a merge without `--delete-branch`)
  leaves the remote behind too. Treat the four-step cleanup as part of the
  merge, not a periodic chore — a long-lived branch list with 8 stale entries
  is how an "already-shipped" branch gets accidentally re-based and re-PR'd.

### Hard-won lessons
- **2026-06-17, TASK-54.1–54.7 (morphogenesis):** running only roborev and
  ignoring `gemini-code-assist` inline reviews let a real HIGH-severity bug
  (`--refresh-interval 0` tight loop) ship through 7 PRs. Bot-review
  handling is part of the merge gate.
- **2026-06-17, TASK-54.8–54.10 (morphogenesis):** left 64 roborev reviews
  open across the series because steps 6/7 were missing from the loop.
  `roborev compact --all-branches` cut them to 15 and surfaced 7 verified
  findings that had been silently accumulating. Always close per-PR and
  compact when open count > 10.
- **2026-06-17, TASK-55.3 (morphogenesis):** left a stray `#[derive]` on
  `init_gpu_resources` during a Python-script extraction. Non-cuda CI
  stayed green (cfg-gated). `greptile-apps[bot]` — the 5th reviewer bot —
  caught it via static analysis (P1). Always poll `pulls/<N>/comments`
  for ALL bot users, not just check for non-empty review summaries.
- **2026-06-17:** across PRs #22–#50, left 23 review threads open after
  replying — the "Resolve conversation" GraphQL step was missing from
  the loop. Added as step 6. Never leave a thread open after triage.
- **2026-06-18, slice-4 PRs (#191/#192 on `2d`):** the review loop was
  fully executed (roborev 2×2 × multiple rounds, all 5 bot reviews read
  including greptile, every thread replied+resolved, all roborev jobs
  closed) — but the PR
  bodies were merged with `[ ]` checkboxes on those steps still
  unchecked. The checkboxes were authored as a to-do list at PR-open
  time and never re-marked `[x]` once the loop completed. To a future
  reader (and the same agent in a later session) a merged PR with
  "`[ ]` roborev 2×2 enqueued" reads as "the loop was abandoned".
  Added pre-merge checklist item #5: re-mark the PR body's review-loop
  checkboxes `[x]` before `gh pr merge`. The body is a status report,
  not a to-do list.
- **2026-06-17, TASK-54.11 (morphogenesis):** codex security and
  `gemini-code-assist` independently caught a behavior regression
  (`is_allowed_snapshot_host` subdomain-matching lost during a refactor).
  Fixed in the same PR rather than follow-up because of step 4. This is
  exactly why step 4 forbids merging with HIGH/MEDIUM outstanding.
- **2026-06-18, post-umbrella cleanup (`2d`):** after merging a 3-PR series
  (#193/#194/#195), `git branch --merged main` reported only `main` itself —
  looked like "nothing is merged". In fact all three feature branches AND three
  older ones (#156/#189/#155) were MERGED via GitHub **squash-merge**, which
  mints a fresh commit SHA on `main`; the feature tip is never in main's
  history, so `--merged` is silently empty. The reliable signal is
  `gh pr list --state merged --head <branch>`. Six stale branches (3 local,
  3 remote) had been sitting for up to a week because of this blind spot.
  Added the Post-merge branch cleanup section above. Never trust `--merged`
  in a squash-merge repo.
- **2026-06-18, TASK-132.8 (`2d` runbook PR #197):** edited task files
  (`task-132.8`, `task-132.5.2.4`, and earlier `task-132.5.2.5` across the
  umbrella series) **directly** — `status` flip, AC checkboxes, AC text
  rewrites, final-summary appends — instead of through the `backlog` CLI.
  The repo's `CLAUDE.md:51` ("NEVER EDIT TASK FILES DIRECTLY. Edit Only via
  CLI") and `AGENTS.md:23` both mandate the CLI; the CLI was installed
  (`backlog 1.47.0`, supports `--check-ac`/`--status`/`--final-summary`/
  `--notes`), so this was not a tooling gap — it was forgetting the rule
  mid-session. greptile caught it (P1). The CONTENT was correct (no revert);
  the violation is procedural — direct edits bypass the CLI's AC-index /
  status-enum validation and can drift a task file's shape. Before editing a
  task file in `backlog/tasks/`, grep for the local "NEVER EDIT TASK FILES"
  rule (CLAUDE.md / repo AGENTS.md) and use the CLI instead:
  `backlog task edit <id> -s "In Progress"`, `--check-ac <N>`,
  `--final-summary "..."`, `--notes "..."`. Repeated across the whole
  session — so this is a real failure mode, not a one-off.

---

## Project-specific

<!-- Replace this section in each consuming repo. Typical contents: -->

### Build & Test Commands
<!-- e.g. cargo / mix / go / pnpm commands -->

### Project Structure
<!-- one-line per top-level directory/component -->

### Project invariants
<!-- any non-negotiable security/privacy/correctness invariants reviewers
     must preserve across changes -->
