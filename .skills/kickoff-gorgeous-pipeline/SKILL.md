---
name: kickoff-gorgeous-pipeline
description: "Runs the whole gorgeous delivery pipeline from the current branch in one invocation: first fast-gorgeous-forward (the interactive window — upstream questions and rebase conflicts are settled with the user up front), then UNATTENDED to the end: prepare-gorgeous-pr with its commit-reshaping offer suppressed, one code-gorgeous-review fleet round, and the-gorgeous-loop until the fleet passes the bar (only excellent/good, more excellent than good, mean >= 4.5). Never pushes and never opens the PR — send-gorgeous-pr stays a separate human step. Use when the user invokes kickoff-gorgeous-pipeline, /kickoff-gorgeous-pipeline, or asks to kick off the gorgeous pipeline."
---

# kickoff-gorgeous-pipeline — fast-forward with the user, then run unattended to a passing fleet

The contract:

> **Stage 1 is the only interactive part: fast-forward the branch onto the freshest upstream main, resolving conflicts with the user. Everything after is unattended: prepare the PR description without asking, run the review fleet, and loop fix-and-review until the bar is met or the loop is stuck. Never push, never open a PR.**

## Stage 0. Preconditions — check FIRST; if any fails, stop and say exactly how to fix it

- **Inside a git repository, on a non-default branch, with a clean working tree.** If any of these fails, stop and say which.

- **The member skills are readable** — `fast-gorgeous-forward`, `prepare-gorgeous-pr`, `code-gorgeous-review`, and `the-gorgeous-loop`, from this repo's `.skills/` or from your harness's global skills. If any is missing, stop and tell the user to install the family: `scsh installskills --global https://github.com/dkorolev/beautiful-skills https://github.com/dkorolev/code-review-skills`.

- **scsh is installed and new enough for global skills (1.25+):** `command -v scsh && scsh help installskills 2>&1 | grep -q -- --global`. If not, tell the user to run `cargo install scsh` and stop. Leave every deeper check (profile resolution, `tmp/` gitignored, route availability) to the member skills — they each verify their own ground.

- **Tell the user the plan before starting:** one line per stage, and that after stage 1 finishes you will run without further questions until done or stuck.

## Stage 1. Fast-forward — the interactive window

- Read and follow the **fast-gorgeous-forward** skill end-to-end. This is where all human interaction belongs: choosing an upstream when none qualifies, and resolving genuine rebase conflicts one question at a time. If the branch is already based on the upstream tip, it says so — proceed.

- **Then bring the local base up to the same tip.** The review stages diff against the LOCAL `main`/`master`, so after the rebase, fast-forward the local base branch to the upstream tip the branch was just replayed onto: `git fetch <upstream> <main>:<main>` (a fast-forward-only update of the un-checked-out branch; never force it). Without this, a stale local `main` would put other people's upstream commits inside the review diff.

- From here on, do not ask the user anything. If a later stage hits a hard stop, report and end — do not improvise around a member skill's stop condition.

## Stage 2. Prepare — unattended

Read and follow the **prepare-gorgeous-pr** skill, with one override for this pipeline:

- **Suppress the offer to factor commits.** Where that skill would lay out a split of oversized or mixed commits and ask whether to proceed, do not ask and do not split — keep the commit structure exactly as it is. Still apply the rules it enforces without asking: `PR-DESCRIPTION.md` must end up as the unique last notes commit (amended in place when it already is), authored by its special notes identity.

Re-runs are safe: on an up-to-date branch the skill amends the existing notes commit rather than stacking a new one.

## Stage 3. First review round — unattended

Read and follow the **code-gorgeous-review** skill, with the same override **the-gorgeous-loop** uses:

- Ignore its final "ask which important cluster to go deeper on — and stop." Collect the full report, write `tmp/code-gorgeous-review.md`, and continue straight to stage 4.

- Honor its hard stops as pipeline stops: no resolvable `code-review` profile, or every invocation skipped or failed (no runnable model route on this host) — report plainly and end.

## Stage 4. The loop — unattended until the bar or stuck

Read and follow the **the-gorgeous-loop** skill end-to-end. It already runs unattended: it checks the stopping bar first (the stage 3 round may already pass), fixes every important cluster without asking, re-runs prepare (same no-reshaping discipline as stage 2) and the review fleet, and repeats. Its practical limit is this pipeline's too: if the same cluster survives three consecutive rounds unchanged, it pauses and explains what is stuck — end there and hand back to the user.

## Stage 5. Report

Write `tmp/kickoff-gorgeous-pipeline.md` and print it: one line per stage with its outcome (the upstream tip fast-forwarded onto, whether prepare amended or created the notes commit, the loop's final score table and iteration count), then the handoff: **nothing was pushed; when ready, `/send-gorgeous-pr` opens the PR** — and if upstream main moved while the loop ran, `/fast-gorgeous-forward` once more first.

## Safety and scope

- All interaction happens in stage 1; a question that would arise later is a stop, not a prompt.

- Local only: never push, never force-push, never open or comment on a PR — that is `/send-gorgeous-pr`, deliberately outside this pipeline.

- Never install `.scsh.yml` or `.skills/` into the target repo; the member skills resolve the fleet from the repo's own manifest or the global one.

- Member skills' own safety rules all stand; this skill only sequences them and suppresses their questions where stated. Scratch under the gitignored `tmp/`; backup refs before any history rewrite come from the member skills themselves.
