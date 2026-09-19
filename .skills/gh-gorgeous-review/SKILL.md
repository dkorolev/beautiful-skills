---
name: gh-gorgeous-review
description: "Review a GitHub PR with the gorgeous fleet, resuming a matching scsh Web UI kickoff when present, then publish by default the findings as one human-voiced GitHub review (COMMENT or APPROVE). Use when the user invokes gh-gorgeous-review, /gh-gorgeous-review, or gives a GitHub pull-request URL to review with the gorgeous fleet. Writes fixed report paths under tmp/; do not run concurrently in the same invocation repository or for the same PR."
---

# gh-gorgeous-review — review a GitHub PR with the gorgeous fleet, then publish the findings

The contract:

> **Given a PR link, build a faithful local replica of the PR under `${SCSH_HOME:-$HOME/.scsh}/github-reviews/pr-<number>-<repo>-<owner>/` — full clone (force-refreshed in place if it already exists), PR feature branch checked out, the PR's actual base branch pinned as local `main`, the real PR description recreated as a fake `PR-DESCRIPTION.md` notes commit — snapshot subscription-model quotas with `scsh quota` before and after, run `/code-gorgeous-review` in that replica, report the quota deltas to the operator in chat only, and tell the user the findings as a flat publication-oriented summary — never as clusters, never with an offer to fix or dig deeper. If — and only if — every reviewer in the fleet succeeded, prepare one review: inline comments anchored to the diff wherever possible, plus a single PR-level summary at the end, all phrased as a gracious human reviewer with zero AI/agent/tooling attribution. Invoking this skill authorizes publication by default; publish without asking again unless the user explicitly requested a preview or local-only review. The review is a COMMENT — unless the grades clear the approval bar (only excellent/good, at least as many excellent as good) and the PR is open and not yet approved by the authenticated user, in which case the very same review is submitted as an APPROVE. Never push code, never request changes, never edit the PR itself.**

The argument is the PR link, e.g. `https://github.com/<owner>/<repo>/pull/<number>`. When run through the `gh-gorgeous-review` profile, read the PR link from `PR_URL`. If neither an argument nor `PR_URL` supplies it, ask for the PR link and stop.

## Review preparation and result files

When a workflow asks only to prepare a review from supplied grades and findings, perform only that preparation step. Use local commands or scripts to read the named input environment variables and `SCSH_RESULT`, combine the findings, and write the required JSON to the path in `SCSH_RESULT`, creating its parent directory if needed. Read only the named inputs, not the full environment. Follow the step's declared output schema; a chat reply does not replace its result file. Do not prohibit all commands: that would prevent the required input and output operations.

Preparation does not clone or refresh a checkout, run git commands, access the network, rerun reviewers, modify repository files other than the result file, or publish to GitHub. The dedicated publication step owns GitHub writes and the publication checks below. These restrictions apply to preparation alone; the full conversational workflow still performs its explicit checkout, review, and publication steps.

## Entry points and browser resume

`gh-gorgeous-review` is the operation's name in both places:

- `$gh-gorgeous-review <PR URL>` (or `/gh-gorgeous-review`) starts and owns the complete conversational workflow.

- `scsh`'s Run page exposes **Start gh-gorgeous-review**. That browser entry prepares the same durable checkout and runs the built-in `gh-gorgeous-review` workflow: quota planning, five specialty reviewers on each eligible harness, one preparation step on the selected publisher harness, publication, and the final quota snapshot. Its eligible routes come from the workflow plan; the conversational entry uses the machine-wide fleet in step 5. It writes `tmp/gh-gorgeous-review-browser.json` in the review checkout, updates its `state` through `running`, `reviewed`, `publishing`, and `published` (or `failed` / `publication_failed`), and stores the before/after snapshots beside it. Browser kickoff publishes one review automatically after validating the complete fleet and the current PR head/base; publication has its own visible job step.

After resolving the PR metadata and clone path in step 1, inspect that browser receipt before refreshing the checkout or starting quotas. A receipt is resumable only when all of these are true:

- `operation` is `gh-gorgeous-review`, `url` identifies this PR, `state` is `reviewed`, and `reviewed_head` equals GitHub's current `headRefOid`.

- The recorded session completed successfully and every expected fleet result exists with a successful route and a parseable grade; use the recorded workflow plan for a built-in browser job, or the machine-wide manifest for a conversational fleet run, to determine the expected routes.

- The checkout is clean, its current HEAD is the single reconstructed-description notes commit whose parent is `reviewed_head`, and its local `main` resolves to `baseRefOid`, matching the receipt's `base_head`.

When the receipt passes, do not clone, refresh, snapshot again, or rerun the fleet. Use the existing result files and quota snapshots, then continue with the quota delta and flat publication summary in steps 6-8. If its state is `running`, link the browser job as `http://127.0.0.1:7274/job/<session>` and stop without starting a duplicate. A `published` receipt is already complete: read `tmp/gh-review-published.json`, link the review, and do not post again. A `publishing` receipt means publication is in progress; link the job and do not duplicate it. A `publication_failed` receipt may reuse the completed fleet after the checks above; reconcile GitHub's existing reviews before retrying, since a lost response may follow a successful POST. A failed, stale, incomplete, or mismatched receipt is not reusable; explain why briefly and follow the ordinary workflow below.

## 1. Parse the argument and pull PR metadata

- Parse `<owner>`, `<repo>`, `<number>` from the URL. Accept the bare `owner/repo#number` form too.

- Fetch the metadata you need in one call:

  ```sh
  gh pr view <URL> --json number,title,body,author,baseRefName,baseRefOid,headRefName,headRefOid,state,url,isDraft
  ```

  If `gh` is not authenticated, tell the user to run `gh auth login` (suggest typing `! gh auth login` so it runs interactively in the session) and stop.

- Also resolve the repo's **default branch** for context: `gh repo view <owner>/<repo> --json defaultBranchRef -q .defaultBranchRef.name`. Local `main` will be pinned to the PR's `baseRefName`, not the default branch. If they differ, call that out in the final report.

- A merged or closed PR is still reviewable; just mention its state in the final report.

## 2. The `pr-<number>-<repo>-<owner>` clone — full, non-shallow, force-refreshed in place

Clone root: `${SCSH_HOME:-$HOME/.scsh}/github-reviews/`. Create it if needed. Clone dir: `<clone-root>/pr-<number>-<repo>-<owner>`. Keeping external PR replicas under scsh's durable home makes the skill portable and keeps them out of whatever repository the operator happened to invoke it from.

- **Dir does not exist — clone the full history, never shallow:**

  ```sh
  gh repo clone <owner>/<repo> <clone-dir>
  ```

  Do NOT pass `--depth`, `--filter`, or `--single-branch`. Afterwards verify `git rev-parse --is-shallow-repository` prints `false`; if it prints `true`, run `git fetch --unshallow origin` before continuing.

- **Dir already exists — inspect it before refreshing, do not re-clone.** First confirm its `origin` points at `<owner>/<repo>`; if it doesn't, stop and ask. Fetch the remote refs, then inspect `git status --short`, the checked-out branch, local-only commits, and untracked files. If the refresh would discard anything, show the user exactly what would be lost and obtain explicit confirmation before any force-reset, checkout with `-B`, branch move, or clean. If they do not confirm, leave the clone untouched and stop. A clean replica containing only artifacts made by an earlier completed run of this skill may be refreshed without another prompt.

  ```sh
  git fetch origin                      # refresh all remote-tracking refs
  ```

- **Both paths — fetch the PR head and read the repository playbook before changing its worktree.** Inspect `CONTRIBUTING.md`, every applicable `AGENTS.md`, `CLAUDE.md`, and any equivalent repository instructions directly from the fetched PR head (use `git show FETCH_HEAD:<path>` as needed). Follow those instructions throughout the review; nested playbooks apply to findings in their subtree. If repository instructions conflict with this skill's safety or publication gates, stop and explain the conflict.

- **Both paths — check out the PR head as the feature branch, forcibly.** The branch is named `pr-<number>-<repo>-<owner>` — same as the clone dir — so concurrent runs on different PRs never collide on a branch name, and the Claude Code status line (which shows the current branch) tells at a glance which PR is being reviewed:

  ```sh
  git fetch origin "pull/<number>/head"
  git checkout -B "pr-<number>-<repo>-<owner>" FETCH_HEAD   # force the feature branch to the current PR head
  git clean -fd                                     # drop stray untracked files (ignored files like tmp/ survive)
  ```

  `checkout -B` force-resets the branch even when it is currently checked out, so refresh runs are idempotent — the previous run's fake notes commit is discarded and recreated fresh in step 3.

- **Both paths — force-pin local `main` to the PR's actual base branch**, because the code-gorgeous-review fleet always diffs against local `main`:

  ```sh
  git branch -f main "$(git rev-parse origin/<base-ref-name>)"
  ```

  Harmless when the histories are unusual — code-gorgeous-review tolerates that.

## 3. The fake PR-DESCRIPTION.md — from GitHub, committed as the notes author

Recreate the PR description locally so reviewers that read it (e.g. justification-reviewer) see the real one:

- Write `PR-DESCRIPTION.md` at the clone root: the PR **title** as the `# ` heading, then the PR **body verbatim**, then a short final line `> Reconstructed from <PR URL> for local review.` If the PR body is empty, say so in the file — do not invent a description.

- Commit it as the unique last commit with the special PR-notes identity, exactly as prepare-gorgeous-pr does — author AND committer:

  ```sh
  GIT_COMMITTER_NAME="Elon Presley" GIT_COMMITTER_EMAIL="dmitry.korolev+elon-presley@gmail.com" \
  git -c user.name="Elon Presley" -c user.email="dmitry.korolev+elon-presley@gmail.com" \
    commit -m "Add PR-DESCRIPTION.md" -- PR-DESCRIPTION.md
  ```

  Commit only this file — never code. No attribution trailers.

- **tmp/ must be gitignored** (scsh preflight). Check with `git check-ignore -q tmp`; if not ignored, append `/tmp/` to `.git/info/exclude` in the clone — this keeps the reviewed diff untouched. Only if scsh's preflight still refuses should you fall back to a separate tiny `.gitignore` commit *before* the notes commit (then recreate the notes commit last).

## 4. Snapshot model quotas — before the review, only for the harnesses the fleet uses

Capture how much quota is left **before** kicking off the fleet, so the operator running this skill (usually inside Claude Code, but it may be a different harness) can see what the review costs. **Snapshot only the harnesses the review fleet actually runs** — currently `claude`, `codex`, and `cursor`, but always derive this set at runtime rather than assuming it (the manifest's routes can change). A bare `scsh quota` queries every configured harness — do not use it. These numbers are for the person at the keyboard only; they never touch the PR (see step 8).

- Derive the fleet's harness set at runtime from the **machine-wide manifest** the fleet will actually run with (step 5). The profile's skill names embed their route (`...-claude-opus-4-8`, `...-codex-terra`, `...-cursor-auto`, ...), so extract the distinct harness tokens:

  ```sh
  scsh list --override-dot-scsh-yml ~/.scsh/.scsh.yml --json 2>/dev/null \
    | python3 -c "import json,sys,re; d=json.load(sys.stdin); names=[s for p in d['profiles'] if p['name']=='code-gorgeous-review' for s in p['skills']]; print('\n'.join(sorted({m.group(1) for n in names for m in [re.search(r'-(claude|codex|cursor|opencode)-', n)] if m})))"
  ```

  (If this yields nothing, fall back to the literal set `claude` + `codex` + `cursor` — the stock fleet contains no other harnesses.)

  If this yields no harnesses (e.g. offline, or probe unavailable), skip quota reporting entirely and say so in chat — it must never block the review.

- Snapshot each harness in that set — one `scsh quota <harness> --json` call per harness (the harness argument comes **before** `--json`). Stash each in the clone's gitignored `tmp/` (never committed):

  ```sh
  for h in <the harnesses from above>; do scsh quota "$h" --json > "tmp/quota-before-$h.json"; done
  ```

  Best-effort per harness: if one call fails, note it and carry on.

- Print a compact "before" snapshot to the user: one row per harness window — harness, plan, window label, `used_percent`, and reset time — read from each `tmp/quota-before-<harness>.json` (`.harnesses[].windows[]`). Each harness's `summary` string is a fine one-liner too.

## 5. Run the gorgeous review

- From inside the clone, invoke the global `/code-gorgeous-review` skill (via the Skill tool) and follow it there with its default base — local `main`, which you pinned to the PR base in step 2. Do not fetch or pull anything further; the replica is complete.

- Let that skill own the mechanics: scsh preflight, the fleet run, collecting the result JSONs, the summary table, and its `tmp/code-gorgeous-review.md` report inside the clone.

- **Use the full machine-wide fleet, including Claude.** Every `scsh run`, `scsh check-profile`, and `scsh list` this skill triggers must pass `--override-dot-scsh-yml ~/.scsh/.scsh.yml`, overriding code-gorgeous-review's plain commands and any repo-local profile. Before the run, verify `scsh check-profile code-gorgeous-review --override-dot-scsh-yml ~/.scsh/.scsh.yml` reports 15 skills and `scsh list --override-dot-scsh-yml ~/.scsh/.scsh.yml --json` shows five `claude-*`, five `codex-*`, and five `cursor-*` routes in that profile. If the machine-wide manifest is missing or does not resolve the full 15-route fleet, tell the user to reinstall it with `scsh installskills --global https://github.com/dkorolev/code-review-skills` and stop.

- **Two overrides to code-gorgeous-review's reporting, which win here:** do NOT group findings into clusters, and do NOT ask its closing "which cluster do you want to go deeper on?" question (or any variant offering to fix, resolve, or investigate further). This skill's output is a review to publish, not a fix-it workflow; findings are summarized flat in step 7 and the run ends after step 8.

## 6. Snapshot model quotas — after the review, and report the deltas

As soon as the fleet run in step 5 returns, re-snapshot the **same harness set from step 4** (not all four) and diff it against the before:

```sh
for h in <the same harnesses as step 4>; do scsh quota "$h" --json > "tmp/quota-after-$h.json"; done
```

- Match windows by `harness` + window `id` across the `tmp/quota-before-<harness>.json` and `tmp/quota-after-<harness>.json` files. For each window report: `before% -> after%`, the delta in percentage points (`delta = after - before` — the quota this review consumed), and how much is left (`100 - after%`) with its reset time. Highlight windows where `delta > 0`; unchanged windows can be collapsed to a single "unchanged" line.

- Give a rough "how many more comparable reviews fit" estimate: for each moved window, `floor((100 - after%) / delta)` more runs of this size before that window's cap, and name the window that runs out first — the binding constraint — alongside when it resets. Caveat it plainly: it's a linear extrapolation from a single sample, and real cost swings with PR size and cache hits.

- If either snapshot is missing (quota was unavailable), just say quota deltas aren't available and move on — never block on it.

- This whole quota report goes to the operator in chat only. Do not write it into `tmp/code-gorgeous-review.md`, any file that could reach the PR, or any PR comment.

## 7. Report to the user — a flat summary for deciding what to leave as comments

- Lead with a one-line header naming the PR (`<owner>/<repo>#<number> — <title>`), its state, and the base ref/SHA reviewed against, then the one-row-per-invocation summary table and the per-reviewer rollup, per the code-gorgeous-review format.

- Then a **flat findings summary — no clusters.** Pool every issue from every successful invocation, dedupe findings that are plainly the same thing raised by multiple routes (keep a "raised by N routes" note), and list one line per distinct finding: `file:line` — one-sentence description — severity — proposed disposition (**inline comment** / **PR-level note** / **drop**, with a word on why for drops, e.g. low-confidence or pure taste). Order by severity, then file. This list IS the decision sheet for step 8: what gets published as an inline comment, what folds into the PR-level body, and what is dropped.

- Write the same flat summary (not clusters) into `tmp/code-gorgeous-review.md` in the clone, and point the user at it.

- Before changing directories, remember the invocation repository. When launched through the `gh-gorgeous-review` profile, scsh preflight guarantees that its `tmp/` is gitignored: always write the complete operator report to `tmp/gh-gorgeous-review.md`, the profile's declared result path, and treat an inability to write it as a failed run rather than silently substituting another location. On a direct skill invocation outside that profile, also write this copy when the invocation repository has a gitignored `tmp/`; otherwise the durable clone report and chat are sufficient.

- Leave the clone in place — the next run of this skill on the same PR reuses and force-refreshes it.

- Do not offer to fix findings, apply suggestions, or go deeper on any of them — end the report with the publication outcome of step 8 and nothing else. If the user wants fixes, they will ask.

## 8. Publish the findings by default — only after a fully successful fleet run

**The gate.** A successful final retry supersedes its earlier failed attempt. Publish only when every reviewer invocation in the fleet **succeeded** — no failed, errored, or skipped rows in the summary table. If even one failed, skip this step entirely, say so in chat, and leave the PR untouched; the user can re-run the skill once the failure is resolved.

**Publication is part of the request.** A `$gh-gorgeous-review` invocation or a browser click on **Start gh-gorgeous-review** authorizes posting the resulting review. Show the flat decision sheet and event (`COMMENT` or `APPROVE`), then publish without another confirmation. An explicit preview, draft-only, local-only, or do-not-post instruction opts out; in that case leave GitHub untouched.

**The approval bar.** The single review is normally `event=COMMENT`, but it becomes `event=APPROVE` when ALL of the following hold:

- Every grade in the fleet is **excellent** or **good**, and there are at least as many excellent as good — i.e. with excellent=5 and good=4, the mean score is >= 4.5. Any other grade anywhere (ok, poor, failed, ...) keeps it a COMMENT.

- The PR is **open** (not merged, not closed). A draft PR also stays a COMMENT — approving drafts is noise.

- The authenticated user has **not already approved** the current state: check with `gh api user -q .login` and `gh pr view <URL> --json latestReviews`; if that login's latest review is `APPROVED`, keep it a COMMENT — do not stack approvals.

- The authenticated user is not the PR author (GitHub rejects self-approval).

When the bar is met, say so explicitly in the chat report ("grades cleared the approval bar — submitting as APPROVE"). Never `REQUEST_CHANGES` under any circumstances. If the `APPROVE` submission is rejected by the API for any reason, retry once as `event=COMMENT` so the findings still land.

Before posting, recheck the PR head and base against the reviewed revision; stop on a mismatch. Check for an existing review from the authenticated user containing `<!-- review-head:<PR head SHA> -->` and link it instead of posting a duplicate. Append that marker to the review body and save the successful API response to `tmp/gh-review-published.json`. On an ambiguous network failure, inspect GitHub before retrying; never blindly repeat the POST.

**One review, two layers.** Post everything as a single pull-request review — `COMMENT` or, per the bar above, `APPROVE`:

- **Inline, as much as possible.** Each finding the step-7 decision sheet marked for publication becomes an inline comment anchored to the exact file and line it concerns. Inline comments can only attach to lines present in the PR diff (`side: RIGHT` for added/changed lines, `side: LEFT` only for deletions); verify each anchor against `gh pr diff` before including it. A finding whose location is not in the diff — or that spans whole files or the PR's overall shape — moves to the PR-level body instead. Never anchor anything to `PR-DESCRIPTION.md`; that file exists only in the local replica.

- **One PR-level comment at the end.** The review's `body` is the single PR-level comment: a brief, friendly overall impression, then any findings that couldn't be anchored inline, then the stylistic notes condensed to a short list. Do not paste the raw summary table, grades, or reviewer names into it.

Submit it in one API call, building the JSON payload in the clone's `tmp/` (e.g. `tmp/gh-review-payload.json`, never committed) to sidestep shell quoting:

```sh
gh api "repos/<owner>/<repo>/pulls/<number>/reviews" --input tmp/gh-review-payload.json
```

with a payload of the shape `{"commit_id": "<PR head SHA>", "event": "<COMMENT or APPROVE>", "body": "<PR-level comment>", "comments": [{"path": "...", "line": N, "side": "RIGHT", "body": "..."}, ...]}`. If the call is rejected because of a bad anchor, drop the offending anchor(s) into the PR-level body and retry once.

**The voice.** Every comment — inline and PR-level — reads as if written by a thoughtful, courteous human colleague:

- Lead with the observation, then why it matters, then a concrete suggestion. Keep each comment self-contained and short; use ```suggestion``` blocks when a small, certain fix fits.

- Warm and collegial, not clinical: acknowledge what's done well, phrase concerns as questions or suggestions where reasonable, and never lecture.

- **Absolutely no AI, agent, or tooling attribution.** No "Generated with Claude Code", no co-author trailers, no robot emoji signatures, no mention of Claude, models, agents, scsh, reviewer fleets, or individual reviewer names — in any comment, anywhere. The model-quota snapshots and deltas from steps 4 and 6 are operator-only diagnostics: never put usage percentages, quota deltas, harness names, plan names, or reset times into any comment.

**Still never:** push code, request changes, approve outside the bar above, edit the PR title/body/labels, or post anything beyond this one review. After posting, link the published review in the chat report and state whether it was a COMMENT or an APPROVE.
