---
name: code-beautiful-review
description: Runs the scsh `code-review` profile (the reviewer fleet) over the current branch, waits for it to finish, then reads every reviewer's result, prints a one-row-per-reviewer summary table (reviewer, duration, rating, issue count), and clusters the pooled findings into labeled groups A/B/C/D with a two-sentence description each, asking which to go deeper on. Read-only: it never edits code, pushes, or opens a PR. Use when the user invokes code-beautiful-review, /code-beautiful-review, or asks to run the code review and group/cluster the comments.
---

# code-beautiful-review — run the review fleet, then cluster the findings

The contract:

> **Run the `code-review` reviewer fleet through scsh, wait for it, then turn its scattered findings into one summary table and a handful of labeled clusters — and stop there, asking the user which cluster to open. Report only; change nothing.**

## 1. Preconditions — check FIRST; if any fails, stop and say exactly how to fix it

- **scsh is installed:** `command -v scsh`. If it is missing, tell the user to install scsh (see https://github.com/dkorolev/scsh for details) and stop — do not improvise a review by hand.

- **The `code-review` profile exists and is non-empty:** run `scsh check-profile code-review` (exit 0 means the profile is present with at least one skill; this is runtime-free). If it is non-zero, tell the user to install the reviewers with `scsh installskills https://github.com/dkorolev/code-review-skills`, then stop.

- **Let scsh enforce the rest.** Its own `run` preflight checks git, that the tree is clean, that `.scsh.yml` is valid, that `tmp/` is gitignored, and that a container runtime is up — each failure names the one fix. If `scsh run` fails preflight, surface that message verbatim and stop.

## 2. Run the fleet and wait

- Record the starting point first: `git rev-parse HEAD` (so you can spot anything the run adds), and note the wall-clock start.

- Run the reviewers and wait for completion, keeping the per-skill run dirs so you can time each one, and teeing the output to your own scratch dir: `SCSH_KEEP_RUNS=1 scsh run code-review 2>&1 | tee tmp/code-beautiful-review-<YYYYMMDD>-<HHMMSS>-<rand>/run.out`. scsh runs every reviewer in parallel, each in its own ephemeral container on a clean clone of the branch.

- Do **not** abort if `scsh run` exits non-zero. Every skill runs regardless; a non-zero exit means at least one reviewer failed to produce its result, but the others' results are still there. Collect what exists and mark the rest FAILED in the table.

## 3. Collect the output

- The authoritative output is each reviewer's **result JSON**, which scsh copies back into the repo on success (any prior file moved aside to `*.bak.<utc>`). Get the reviewer list and each declared `result` path from `scsh list` (or `.scsh.yml`) for the `code-review` profile — by convention `tmp/code-review-<skill>.json`. Each file has the shape `{ result: { grade, issues_found }, issues: [ { commit, file, line, description, suggestion } ] }`, where `grade` is one of `excellent | good | average | poor` and `issues_found` equals `issues.length`.

- If the reviewers are configured for commit delivery (`commits: true` in `.scsh.yml`), the same findings also land as new commits on the branch authored by the dedicated review account — everything in `git log <starting-HEAD>..HEAD` by that author. Read those too and correlate them with the JSON, but treat the JSON as the source of truth.

- **Per-reviewer duration:** from the kept run dirs (`SCSH_KEEP_RUNS=1` left them in the system temp dir as `scsh-*-run-<skill>/`), read each `tmp/scsh-run.log` and take its first-to-last timestamp. If a log is unavailable, report the overall wall-clock for the run and mark that reviewer's cell `n/a (parallel)`. Remove the kept run dirs once you have read them.

## 4. The summary table — one row per reviewer

Print a table with exactly these columns, one row per skill in the `code-review` profile:

| Reviewer | Duration | Rating | Issues |

- **Reviewer** — the skill name.

- **Duration** — its wall-clock from step 3.

- **Rating** — `result.grade` from its JSON.

- **Issues** — `result.issues_found` from its JSON.

A reviewer whose result file is absent is `FAILED` — say so in its row rather than guessing.

Write the full summary — this table plus the lettered clusters from the next step — to `tmp/code-beautiful-review.md` so the run leaves a persistent report.

## 5. Cluster the findings

- Pool every issue from every reviewer into one list. Group similar ones into clusters — by shared file and line region, by shared root cause or theme, and by the same problem raised by more than one reviewer (overlap is expected here; collapse those duplicates into a single cluster rather than listing each). Label the clusters `A`, `B`, `C`, `D`, and so on.

- For each cluster, give **exactly two sentences**: the first says what the cluster is, the second says where and how it shows up (which files, which reviewers raised it). Then ask the user which cluster they want to go deeper on — and stop. You do not fix, stage, or resolve anything; this skill reports, and a human decides.

## Safety and scope

- Read and report only. The scsh run is a local, sandboxed read of the branch and takes no outward action; you take none either — never edit code, never commit findings, never push, never open or comment on a PR. All scratch you write goes under the gitignored `tmp/`.
