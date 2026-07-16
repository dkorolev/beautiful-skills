---
name: kickoff-gorgeous-pipeline
description: "Runs the whole gorgeous delivery pipeline from the current branch in one invocation: first fast-gorgeous-forward (the interactive window — upstream questions and rebase conflicts are settled with the user up front), then UNATTENDED to the end. On scsh 1.28+ the unattended part IS the scsh-native `gorgeous-pipeline` workflow — prepare, the 15-route Opus/Codex/Cursor fleet, and the fix-review loop run as ONE job whose cycles render live in the session browser; on older scsh it falls back to driving prepare-gorgeous-pr, code-gorgeous-review, and the-gorgeous-loop in-session until the fleet passes the bar (only excellent/good, more excellent than good, mean >= 4.5). Never pushes and never opens the PR — send-gorgeous-pr stays a separate human step. Use when the user invokes kickoff-gorgeous-pipeline, /kickoff-gorgeous-pipeline, or asks to kick off the gorgeous pipeline."
---

# kickoff-gorgeous-pipeline — fast-forward with the user, then run unattended to a passing fleet

The contract:

> **Stage 1 is the only interactive part: fast-forward the branch onto the freshest upstream main, resolving conflicts with the user. Everything after is unattended — and on scsh 1.28+ it is scsh-native: the `gorgeous-pipeline` workflow runs prepare, the review fleet, and the fix-review loop as one job, rendered with cycles in the browser. Never push, never open a PR.**

## Stage 0. Preconditions — check FIRST; if any fails, stop and say exactly how to fix it

- **Inside a git repository, on a non-default branch, with a clean working tree.** If any of these fails, stop and say which.

- **scsh is installed and new enough for global skills (1.25+):** `command -v scsh && scsh help installskills 2>&1 | grep -q -- --global`. If not, tell the user to run `cargo install scsh` and stop.

- **Pick the engine for the unattended stages, and remember the choice:**
  `scsh help def 2>&1 | grep -q "gorgeous-pipeline"` — when it matches (scsh 1.28+), the unattended stages run as the **native `gorgeous-pipeline` workflow** (stage 2); otherwise they run **in-session** (stage 2-fallback). Only the fallback needs the member skills `prepare-gorgeous-pr`, `code-gorgeous-review`, and `the-gorgeous-loop` to be readable; `fast-gorgeous-forward` is required on both paths. If a needed skill is missing, stop and tell the user to install the family: `scsh installskills --global https://github.com/dkorolev/beautiful-skills https://github.com/dkorolev/code-review-skills`.

- **Tell the user the plan before starting:** one line per stage, WHICH engine the unattended part will use, and that after stage 1 finishes you will run without further questions until done or stuck.

## Stage 1. Fast-forward — the interactive window

- Read and follow the **fast-gorgeous-forward** skill end-to-end. This is where all human interaction belongs: choosing an upstream when none qualifies, and resolving genuine rebase conflicts one question at a time. If the branch is already based on the upstream tip, it says so — proceed.

- **Then bring the local base up to the same tip.** The review stages diff against the LOCAL `main`/`master`, so after the rebase, fast-forward the local base branch to the upstream tip the branch was just replayed onto: `git fetch <upstream> <main>:<main>` (a fast-forward-only update of the un-checked-out branch; never force it). Without this, a stale local `main` would put other people's upstream commits inside the review diff.

- From here on, do not ask the user anything. If a later stage hits a hard stop, report and end — do not improvise around a member skill's stop condition.

## Stage 2. The native pipeline — scsh runs the loop (scsh 1.28+)

The `gorgeous-pipeline` builtin IS the unattended pipeline, run by scsh itself: `prepare` writes or updates `PR-DESCRIPTION.md` from the branch's own history, the 15-route fleet (conventions, justification, reviewability, sanity, and testing — each independently through Opus 4.8, Codex Terra, and Cursor Auto) reviews the branch's change set once, and then decide → fix → 15 reviewers loops until the bar is met (15/15 succeed, all excellent/good, excellent > good, mean ≥ 4.5). The whole loop is ONE job whose cycles render live on its page.

- The repo must be run-ready (`tmp/` or `.harness/tmp` gitignored, tree committed) — `scsh run` verifies this itself and says exactly what is missing; relay its message and stop if it refuses.

- Run it, capturing the output: `scsh run --def gorgeous-pipeline 2>&1 | tee tmp/kickoff-gorgeous-pipeline-run.log`. As soon as scsh prints the session-browser link, tell the user: the job page shows the loop's cycles, every reviewer's recording, and the fleet tables, live.

- Wait for the run to finish. `prepare` and every in-loop `fix` come back as commits on the current branch (scsh's commit-integration; the diffs are browsable from the job page). Reviewers are read-only.

- Read the outcome from the run output and the result files it names: the final `decide` breaking the loop means the bar was met; a run that ends without approval (loop ceiling, a failed route, every route skipped) is reported exactly as scsh describes it. Do not re-run or improvise around a refusal.

## Stage 2-fallback. In-session pipeline (scsh without the `gorgeous-pipeline` def)

Unchanged from the original pipeline, three sub-stages, all unattended:

- **Prepare:** read and follow **prepare-gorgeous-pr**, suppressing its offer to factor commits — do not ask and do not split; still enforce its rules (`PR-DESCRIPTION.md` as the unique last notes commit, authored by its special notes identity).

- **First review round:** read and follow **code-gorgeous-review**, ignoring its final "ask which important cluster to go deeper on — and stop": collect the full report, write `tmp/code-gorgeous-review.md`, and continue. Honor its hard stops as pipeline stops.

- **The loop:** read and follow **the-gorgeous-loop** end-to-end. It checks the stopping bar first, fixes every important cluster without asking, re-runs prepare (same no-reshaping discipline) and the fleet, and repeats. If the same cluster survives three consecutive rounds unchanged, it pauses and explains what is stuck — end there and hand back to the user.

## Stage 3. Report

Write `tmp/kickoff-gorgeous-pipeline.md` and print it: one line per stage with its outcome (the upstream tip fast-forwarded onto, which engine ran the unattended part, the final score table and iteration count — for the native run, as read from the job's results), then the handoff: **nothing was pushed; when ready, `/send-gorgeous-pr` opens the PR** — and if upstream main moved while the loop ran, `/fast-gorgeous-forward` once more first. After a NATIVE run, also say plainly: the pipeline's `prepare` and `fix` commits are authored by scsh's bot identity, so `/send-gorgeous-pr` will offer to reauthor them before pushing — expected, not a problem.

## Safety and scope

- All interaction happens in stage 1; a question that would arise later is a stop, not a prompt.

- Local only: never push, never force-push, never open or comment on a PR — that is `/send-gorgeous-pr`, deliberately outside this pipeline.

- Never install `.scsh.yml` or `.skills/` into the target repo; the native def is built into scsh, and the fallback resolves the fleet from the repo's own manifest or the global one.
