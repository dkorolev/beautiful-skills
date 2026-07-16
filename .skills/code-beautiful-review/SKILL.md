---
name: code-beautiful-review
description: "Runs the scsh `code-review` profile — resolved from the target repo's own .scsh.yml or, when it is not declared there, from the machine-wide manifest installed by `scsh installskills --global` — over the current branch against local main/master or a local base the user names, without writing anything into the target repo. Reads every reviewer's result, prints a one-row-per-invocation summary table, and clusters important findings separately from stylistic comments. Read-only: it never edits code, pushes, or opens a PR. Use when the user invokes code-beautiful-review, /code-beautiful-review, or asks to run the beautiful code review."
---

# code-beautiful-review — run the review fleet, then cluster the findings

The contract:

> **Confirm the `code-review` profile resolves (`scsh check-profile code-review`); review against local `main`/`master` or the local base the user names; run the fleet with plain `scsh run code-review`; wait for it; then report the important findings first and stylistic comments separately. Report only; change nothing. Never write `.scsh.yml` or `.skills/` into the target repo.**

## Where the profile comes from

scsh resolves the `code-review` profile on its own: the target repo's `.scsh.yml` wins when it declares the profile; otherwise scsh falls back to the machine-wide manifest at `~/.scsh/.scsh.yml` (with a one-line note saying so). Either way the skill bodies are injected into the run clone only — the target repository never needs, and never receives, a `.scsh.yml` or `.skills/` of its own. The machine-wide manifest is a one-time setup:

```sh
cargo install scsh
scsh installskills --global
```

## 1. Preconditions — check FIRST; if any fails, stop and say exactly how to fix it

- **scsh is installed:** `command -v scsh`. If it is missing, tell the user to run `cargo install scsh` (see https://github.com/dkorolev/scsh for details) and stop — do not improvise a review by hand.

- **The `code-review` profile resolves and is non-empty:**

  ```sh
  scsh check-profile code-review
  ```

  Exit 0 means scsh found the profile — in this repo's `.scsh.yml` or in the global manifest — with at least one skill. If non-zero, tell the user to run `scsh installskills --global` (scsh 1.25+) and stop. Do **not** run `scsh installskills` into the target repo.

- **Local git repo with a local base branch.** Do not require `origin`, do not require GitHub, do not fetch, and do not require local `main` to match any remote. Use local `main` as the default base, falling back to local `master`; if neither exists, stop and ask for the local branch/ref to compare against. If the user names a base branch/ref explicitly, honor it verbatim.

- **`tmp/` must be gitignored** (scsh preflight). If it is not, stop and tell the user to add `/tmp` (or `tmp/`) to `.gitignore` and commit that — do not install skills.

- **No PR-description gate.** Do not require `PR-DESCRIPTION.md`, do not read it as authoritative context, and do not complain if it is missing, stale, incomplete, or out of date. Review the code diff itself.

- **Route availability is scsh's job — do not hand-roll auth checks.** `scsh run` probes every configured harness·model route itself, prints `⚠ skipping …` for each unavailable one, runs the rest in parallel, and fails only when none remain. If the run reports that every invocation was skipped, stop and tell the user to log in to at least one agent CLI on this host.

- **Let scsh enforce its own runtime preflight only.** Do not add extra freshness, PR-shape, remote, or default-branch ancestry checks. If `scsh run` fails preflight, surface that message verbatim and stop.

## 2. Pin the review base — local main/master, or an explicit local ref

**Host-only local git prep; push IN, pull OUT.** Do not fetch or pull from remotes in this skill. scsh pushes a complete local clone into each container (bind-mount). Reviewers inside containers MUST NOT fetch or pull — they use only refs already in that clone. After containers exit, scsh pulls OUT result JSON. Your job is to bake the chosen local base into what scsh clones *before* `scsh run`.

Each reviewer diffs `origin/main..HEAD` inside its own clone, so make `origin/main` in the reviewed clone point at the chosen local base. Do not reject stale, diverged, forked, or unrelated histories; this skill is allowed to compare whatever HEAD is against the local base branch the user has in this repo. A larger or unusual diff is acceptable.

- **Default base.** Use local `main` if it exists, else local `master`. Resolve the SHA with `git rev-parse <base>`. Do not inspect `origin/main`, do not inspect upstream remotes, and do not run `git fetch`.

- **Explicit base.** If the user asked to review against a specific branch or ref, honor it verbatim as a local ref. Do not fetch it; if it is missing locally, stop and ask for a local ref that exists.

- **Unrelated histories are okay.** If `git merge-base <base> HEAD` fails, still continue. The review is a pragmatic comparison between `<base>` and `HEAD`, not a PR-readiness gate.

**Make the fleet see that base as `origin/main`.** The reviewers take no base argument and always diff `origin/main..HEAD`, so run in a prepared clone under `tmp/` unless this repo's local `main` already points at the chosen base. In the prepared clone, leave the reviewed branch checked out at HEAD and force local `main` to the chosen base SHA with `git branch -f main <base-sha>`. When scsh re-clones this prepared dir, that local `main` becomes the reviewers' `origin/main`, so their diff is exactly local-base-vs-HEAD. The prepared clone is throwaway scratch; this repo's own working tree and refs are never touched.

## 3. Run the fleet and wait

- Run in the directory you settled on in step 2 — this repo if local `main` already was exactly the chosen base, otherwise the prepared clone. Everything below happens there.

- **Containers never fetch.** `scsh run code-review` bind-mounts a host-prepared clone into each container. The reviewer agents must not `git fetch`, `git pull`, or `git clone` — if a reviewer tries, that violates the skill contract. Missing refs are a host prep bug (step 2), not something to fix inside the container.

- Record the starting point first: `git rev-parse HEAD` (so you can spot anything the run adds), and note the wall-clock start.

- Run the reviewers and wait for completion, keeping the per-skill run dirs so you can time each one, and teeing the output to your own scratch dir:

  ```sh
  SCSH_KEEP_RUNS=1 scsh run code-review 2>&1 \
    | tee tmp/code-beautiful-review-<YYYYMMDD>-<HHMMSS>-<rand>/run.out
  ```

  scsh runs every configured invocation in parallel, each in its own ephemeral container on a clean clone of the branch, diffed against the base you pinned in step 2. Unavailable harnesses or models print `⚠ skipping …` and are not run.

- Do **not** abort if `scsh run` exits non-zero. Every configured invocation is attempted; a non-zero exit means at least one reviewer failed to produce its result, but the others' results are still there. Collect what exists and mark the rest `FAILED` or `SKIPPED` in the table.

- **Success for this skill:** after the run, at least one reviewer invocation produced its result JSON. That is enough — you do not need the full matrix. If every invocation was skipped or failed, say so plainly and stop before clustering.

## 4. Collect the output

- The authoritative output is each reviewer's **result JSON**, which scsh copies back into the run directory on success (any prior file moved aside to `*.bak.<utc>`). By convention the files are `tmp/code-review-<invocation>.json` (for example `tmp/code-review-conventions-reviewer-codex-terra.json`, `tmp/code-review-sanity-reviewer-claude-opus-4-8.json`, …), relative to wherever the fleet ran (this repo, or the prepared clone). The definitive invocation list is in the `scsh run` output you teed — one line per invocation, with `⚠ skipping …` marking the `SKIPPED` rows (when the profile lives in this repo's own `.scsh.yml`, `scsh list` shows the same list up front). Each file has the shape `{ result: { grade, issues_found }, issues: [ { commit, file, line, description, suggestion } ] }`, where `grade` is one of `excellent | good | average | poor` and `issues_found` equals `issues.length`.

- **Per-reviewer duration:** from the kept run dirs (`SCSH_KEEP_RUNS=1` left them in the system temp dir as `scsh-*-run-<skill>/`), read each `tmp/scsh-run.log` and take its first-to-last timestamp. If a log is unavailable, report the overall wall-clock for the run and mark that reviewer's cell `n/a (parallel)`. Remove the kept run dirs once you have read them, and delete the prepared clone (if you made one) once you have collected its results.

## 5. The summary table — one row per invocation

Print a table with exactly these columns, one row per skill invocation in the `code-review` profile (up to fifteen with the stock three-route matrix):

| Reviewer | Model route | Duration | Rating | Issues |

- **Reviewer** — the base reviewer (`conventions-reviewer`, `justification-reviewer`, …), parsed from the invocation name before the route suffix.

- **Model route** — the invocation-name suffix after the reviewer name, rendered readably: `codex-terra` as `Codex Terra`, `claude-opus-4-8` as `Opus-4.8`, `cursor-auto` as `Cursor Auto`, and so on for whatever routes the manifest declares.

- **Duration** — its wall-clock from step 4.

- **Rating** — `result.grade` from its JSON.

- **Issues** — `result.issues_found` from its JSON.

A row whose harness was skipped by scsh is `SKIPPED` — say so rather than guessing. A row that ran but whose result file is absent is `FAILED`.

After the full table, print a short **per-reviewer rollup** for the five base reviewers: for each, list how many model routes ran, the issue counts per route, and the strictest grade seen across routes.

Write the full summary — this table, the per-reviewer rollup, and the important/stylistic clusters from the next step — to `tmp/code-beautiful-review.md` in **this** repo (not the prepared clone, which is about to be deleted), so the run leaves a persistent report where the user will look for it. State at the top of the report which local base ref/SHA the review used and whether histories were related.

## 6. Filter and cluster the findings

- Pool every issue from every successful invocation into one list. When tagging an issue for clustering, record which **invocation** raised it (reviewer + model route). Group similar ones into clusters by shared file and line region, root cause, or repeated finding across reviewers/models.

- **Important findings first.** Only promote findings that matter for correctness, runtime behavior, data loss, migrations, concurrency, security, API compatibility, tests that would miss a real regression, or maintainability problems likely to become bugs. Drop low-confidence noise rather than making the user chase it.

- **No style nitpicks in the main findings.** Formatting, naming, comment wording, minor refactors, personal taste, and "could be cleaner" observations do not belong in the important findings unless they hide a real bug. If a stylistic point is worth preserving, put it under a separate section titled `Stylistic Comments` and mark it explicitly as optional.

- For each important cluster, give **exactly two sentences**: the first says what the cluster is, the second says where and how it shows up (which files, which reviewers and model routes raised it). Then ask the user which important cluster they want to go deeper on — and stop. You do not fix, stage, or resolve anything; this skill reports, and a human decides.

## Safety and scope

- Read and report only. The prepared clone and all run scratch are throwaway under the gitignored `tmp/`; the scsh run is a local, sandboxed read of the branch. You take no outward action — never edit code, never commit findings, never push, never open or comment on a PR, and never fetch/pull as part of this skill. This repo's own working tree and refs are left exactly as you found them — **including never writing `.scsh.yml` or `.skills/` into it**.
