---
name: the-beautiful-loop
description: "Loops after code-beautiful-review: applies clustered review fixes (logged in tmp/the-beautiful-loop-fixes.md, never committed), commits (amending or reshaping when sensible), re-runs prepare-beautiful-pr and code-beautiful-review until every reviewer succeeds with only excellent or good grades, more excellent than good, and a mean score of 4.5 or above (excellent=5, good=4). Reviews run against the local base with nothing installed into the target repo (scsh resolves the code-review profile from the repo's .scsh.yml or the global manifest); rebasing onto the freshest upstream main is deferred until after the loop converges. Respects prior fixes, human guidance, and repo maxims. Use when the user invokes the-beautiful-loop, /the-beautiful-loop, or asks to keep fixing review findings until the fleet passes."
---

# the-beautiful-loop — fix review clusters, then loop until the fleet passes

The contract:

> **Read the latest code-beautiful-review results. If the stopping bar is met, declare done. Otherwise fix every important cluster, commit cleanly, re-run prepare-beautiful-pr and code-beautiful-review, and repeat until the bar is met or you are blocked. Never install `.scsh.yml` or `.skills/` into the target repo, and never gate an iteration on the branch sitting on top of the freshest main — proofread your own code first; rebase once, after the loop.**

This is the step **after** `/code-beautiful-review` (and any human follow-up). It closes the loop that `code-beautiful-review` deliberately leaves open.

## Pairing with `/code-beautiful-review`

All fleet runs go through the **code-beautiful-review** skill. It drives plain `scsh run code-review` — scsh resolves the profile from this repo's `.scsh.yml` or from the machine-wide manifest installed by `scsh installskills --global`, and injects the reviewer skill bodies into the run clone only. Do **not** drive scsh yourself outside that skill, and do **not** install skills into the target repo as a fallback. When this skill says "re-run the review," it means: read and follow **code-beautiful-review** end-to-end.

## The base is pinned once — no per-iteration freshness gate

The loop can run for a long time, and upstream `main` may move while it does. That is fine and is **not** this skill's problem: the whole loop reviews and fixes against the **same local base** `code-beautiful-review` uses (local `main`/`master`, or the ref the user named), pinned on the first iteration. Do not fetch, do not compare against `origin/main`, and do not stop an iteration because the branch no longer sits on the freshest main — proofreading your own code comes first; rebasing (`/fast-beautiful-forward`) happens once, after the loop declares done, typically right before `/send-beautiful-pr`.

## Fix log — `tmp/the-beautiful-loop-fixes.md`

Maintain a **separate running log** of what this skill is fixing. Path is fixed:

```
tmp/the-beautiful-loop-fixes.md
```

- **Create or append** at the start of each fix pass (step 3). For each cluster: the finding, files touched, approach taken, and whether it recurred from a prior round.
- **Read it** on later iterations so you do not contradict or duplicate work already logged.
- **Never commit it.** It lives under gitignored `tmp/` — do not `git add`, stage, amend, or rebase it into any commit. Before every commit in step 4, confirm `git status --porcelain` does not list this file as staged; if it is staged, unstage immediately (`git restore --staged tmp/the-beautiful-loop-fixes.md`).
- **Never push it.** This file is local scratch only; `/send-beautiful-pr` will refuse if it appears in the branch history.

The iteration report (`tmp/the-beautiful-loop.md`) stays separate — scores and loop metadata there, fix narrative in the fix log.

## 0. Stopping criteria — check at the TOP of every loop iteration

Before fixing anything, load the **most recent** review round:

- Prefer the JSON result files from the last `/code-beautiful-review` run (`tmp/code-review-*.json`, relative to wherever the fleet ran — usually this repo).
- Fall back to `tmp/code-beautiful-review.md` if JSON is incomplete.

Build the same invocation table `code-beautiful-review` uses: one row per configured invocation in the `code-review` profile (five reviewers times the manifest's model routes — up to fifteen with the stock three-route matrix).

**Grade scores:** `excellent = 5`, `good = 4`, `average = 3`, `poor = 2`.

**Stop — declare done — when ALL of the following hold:**

1. **No failures.** Every invocation that scsh attempted produced a result JSON. Rows marked `FAILED` (ran but no JSON) block stopping. `SKIPPED` rows (harness/model unavailable) do not count toward the score but also do not count as successes — if any configured route was skipped because it was unavailable at run time, treat the round as incomplete and continue fixing; only stop when every route that *can* run did run and succeeded.

2. **Only good or excellent.** Every successful result has `grade` ∈ `{excellent, good}`. Any `average` or `poor` blocks stopping.

3. **More excellent than good.** `count(excellent) > count(good)`.

4. **Mean score ≥ 4.5.** Average of scores over all successful invocations must be **4.5 or above** (not merely 4.0).

When all four hold, print a short **done** report (final table, mean score, excellent vs good counts) and **stop**. Remind the user that the loop never rebased: if upstream main moved while it ran, `/fast-beautiful-forward` is the next step before `/send-beautiful-pr`. Do not enter the fix loop.

If any criterion fails, continue to step 1.

## 1. Preconditions — check FIRST on the first invocation only

- **Inside a git repository.** If not, stop.

- **`tmp/` is gitignored** (scsh preflight for the review re-runs). If not, stop and tell the user to add `/tmp` or `tmp/` to `.gitignore` and commit that — do not install skills into the repo.

- **A prior review exists.** `tmp/code-beautiful-review.md` or at least one `tmp/code-review-*.json` from a recent `/code-beautiful-review` run must be present. If neither exists, stop and tell the user to run `/code-beautiful-review` first.

- **The review skill is available.** The **code-beautiful-review** skill of this family must be readable — in this repo's `.skills/` or as a global skill of your harness. If it is missing, stop.

- **Read the conversation from the top.** Scan the full chat for: prior fixes already applied in this branch, explicit human preferences ("leave X alone", "prefer Y", "document more"), and anything that must not be contradicted. Carry that forward.

## 2. Collect clusters to fix

From the latest review round (JSON + `tmp/code-beautiful-review.md`):

- Take **every important cluster** from the report — not stylistic comments unless they hide a real bug or the same stylistic issue has appeared in multiple consecutive rounds (then treat it as a documentation/clarity fix).

- Merge duplicate clusters across reviewers/models. Note which invocations raised each cluster.

- **Do not ask the user which cluster to tackle.** Unlike `code-beautiful-review`, this skill fixes all important clusters in one pass. If the human already answered a question in chat, that answer is binding.

## 3. Fix — carefully, without contradicting prior work

Apply fixes for every cluster from step 2.

**Fixing principles:**

- **Do not contradict previous fixes** in this branch or session. When in doubt, extend or refine — do not revert.

- **Lean toward the top-level problem.** Prefer fixes that address root cause over local patches that paper over symptoms.

- **Respect human guidance.** If the human said something in chat, follow it even when a reviewer suggests otherwise.

- **Recurring themes → simplify and over-document.** When the same concern appears across multiple review rounds, simplify the code *and* add clear comments or docstrings — err on the side of explaining *why*, not just *what*.

- **Check maxims and similar code.** Before inventing a pattern, read relevant files under `maxims/` (`.md` bodies, scoped by topic) and grep the codebase for how the same problem is handled elsewhere. Match existing conventions; do not reinvent the wheel.

- **Scope discipline.** Fix what the clusters require. Do not refactor unrelated code.

When a cluster is ambiguous, choose the fix that best solves the underlying issue without undoing earlier intentional choices visible in the chat or commit history.

**After applying fixes**, append a short entry per cluster to `tmp/the-beautiful-loop-fixes.md` before committing.

## 4. Commit — sober look at the stack

Working tree must end clean.

- Review every commit in `base..HEAD` (the pinned local base: local `main`, else `master`, or the ref the user named).

- Make an **educated decision** on how to land the fixes:
  - **Amend** when the fix clearly belongs to the commit that introduced the issue and that commit is still yours / not shared upstream.
  - **New commit(s)** when the fix is logically separate or amending would mix unrelated concerns.
  - **Rearrange** (soft-reset and re-commit, or interactive rebase) when the stack is messy and a cleaner partition helps reviewers — same ironclad rule as `prepare-beautiful-pr`: **final code tree byte-identical** except for intentional fix changes; save a backup ref first.

- Never rewrite commits that already exist on a shared remote unless the user explicitly asked for force-push in this conversation (they almost never will here).

- Match the repository's commit message style.

- **Fix log stays out of git.** Confirm `tmp/the-beautiful-loop-fixes.md` is not staged and does not appear in `git diff --cached`. No commit in `base..HEAD` may add or modify that path.

## 5. Re-run prepare-beautiful-pr

Read and follow the **prepare-beautiful-pr** skill.

This recreates `PR-DESCRIPTION.md` and places the Elon Presley notes commit last. Let it reshape commits only when its own rules say to; do not skip it.

**Override for this skill:** `prepare-beautiful-pr` normally requires the base to be an ancestor of HEAD, preferring `origin/main`, and sends you to `/fast-beautiful-forward` when it is not. Inside the loop, skip that gate: hand it the **same pinned local base** the review uses, do not fetch or consult `origin/main`, and do not leave the loop to rebase. If the pinned base is not an ancestor of HEAD, proceed anyway — the description and commit shaping still apply to `base..HEAD`. Rebasing onto the freshest main is a one-time step after the loop, not a per-iteration chore.

## 6. Re-run code-beautiful-review

Read and follow the **code-beautiful-review** skill.

**Override for this skill:** ignore step 6's "ask which cluster to go deeper on — and stop." Collect the full report and clusters, write `tmp/code-beautiful-review.md`, but **do not stop for human cluster selection** — immediately go back to **step 0** (stopping criteria at the top).

## 7. Loop

```
┌─────────────────────────────────────┐
│ 0. Stopping criteria met? → DONE    │
└──────────────┬──────────────────────┘
               │ no
               ▼
┌─────────────────────────────────────┐
│ 2–4. Fix clusters → commit          │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│ 5. prepare-beautiful-pr             │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│ 6. code-beautiful-review            │
└──────────────┬──────────────────────┘
               │
               └──────► back to 0
```

Keep looping until step 0 declares done.

**Practical limits:** if the same cluster survives three consecutive rounds unchanged, pause and tell the user what is stuck and why — do not spin forever. Otherwise keep going.

## 8. Report (each iteration and on done)

Append to `tmp/the-beautiful-loop.md`:

- iteration number;
- stopping-criteria checklist (pass/fail per bullet);
- clusters fixed this round (detail lives in `tmp/the-beautiful-loop-fixes.md`);
- commit strategy used (amend / new / rearrange);
- pointer to latest `tmp/code-beautiful-review.md`.

On **done**, print the final score table and mean, say plainly that the review loop finished, and remind the user to `/fast-beautiful-forward` before `/send-beautiful-pr` if upstream main moved while the loop ran.

## Safety and scope

- **Local only by default.** Fix, commit, and re-review on the branch. Do not push or open a PR — that is `/send-beautiful-pr`.

- **Never contradict prior session fixes or human answers.**

- **Stopping is numeric and strict:** all succeeded, only good/excellent, more excellent than good, mean ≥ 4.5.

- **The base is pinned; freshness is out of scope.** No fetch, no `origin/main` comparison, no per-iteration on-top-of-main gate — rebasing is a single post-loop step.

- **`tmp/the-beautiful-loop-fixes.md` is never committed or pushed.** Local fix log only.

- **Never pollute the target repo** with `.scsh.yml` / `.skills/` installs — reviews always go through `/code-beautiful-review`, and scsh resolves the profile from the repo's own manifest or the global one.

- Scratch under gitignored `tmp/`. Backup refs before any history rewrite.
