---
name: prepare-gorgeous-pr
description: "Run after a feature is built (for example with big-beautiful-build) to get the branch PR-ready. Confirms a clean feature branch on top of main, offers to factor oversized or mixed product commits, then mandatorily removes every earlier PR-DESCRIPTION.md change regardless of author or placement, regenerates the description from the actual code, and verifies one description-only final commit authored and committed by Elon Presley. The description uses only Summary, What This Changes, and Implementation Details, with no test plan or checkboxes. It never pushes or opens the PR. Use when the user invokes prepare-gorgeous-pr, /prepare-gorgeous-pr, or asks to shape a branch and write its PR description."
---

# prepare-gorgeous-pr — shape the commits, then write the PR description

The contract:

> **Get the branch into PR shape — a clean stack of focused product commits on top of main followed by exactly one PR-DESCRIPTION.md-only commit from the special notes author — without ever pushing or opening the PR.**

## 1. Preconditions — check FIRST; if any fails, stop and say exactly how to fix it

- **Inside a git repository.** If not, stop.

- **Repository playbook read.** Before inspecting or rewriting history, read the target repository's `AGENTS.md` files and contribution guidance, including `CONTRIBUTING.md` when present. Follow their conventions for commits, generated files, validation, and scratch data throughout this skill.

- **Working tree clean.** No uncommitted or staged changes. If the tree is dirty, stop and tell the user to commit or stash first — this skill reshapes history and must start from a clean state.

- **Not on the default branch.** Determine the default branch (assume `main`, fall back to `master`). If HEAD is on it, stop: a PR needs a feature branch, not main.

- **Commits on top of main.** Pick the base — the default branch, preferring `origin/main` when a remote exists, otherwise local `main` — and require that the base is an ancestor of HEAD with at least one commit in `base..HEAD`. If there are no commits on top of the base, there is nothing to prepare; stop. If the base is NOT an ancestor (the branch has diverged from main or sits behind it), stop and tell the user to run `/fast-gorgeous-forward` first, so the branch is rebased into a clean, fast-forwardable stack on top of the freshest main.

## 2. Analyze the commits, and offer to factor them down

- Read every commit in `base..HEAD` thoroughly: how large each one is, and whether it is a single coherent change or bundles logically separate concerns — for example a feature plus an unrelated refactor plus test scaffolding all in one commit, or one giant commit for the entire feature.

- Audit `PR-DESCRIPTION.md` across every commit in `base..HEAD`, regardless of commit message or identity. Inspect each commit's actual changed paths rather than relying on a path-limited `git log`, whose history simplification can omit changes later reverted. Read the file at `HEAD` when present and read every historical version before rewriting anything. These versions are prior context only: commit position may suggest chronology, but the final product tree and full code diff are authoritative.

- If any commit is oversized, or groups separate things together, you MUST OFFER to factor the change into several focused commits. Lay out concretely how you would split it — the proposed commits, in order, each a single logical unit — and ask the user whether to proceed. If the user declines, leave the commits as they are and go to step 3.

- If the user agrees, split the work into several commits, each a coherent logical unit in a sensible order. The one ironclad rule is that the final product tree MUST stay byte-identical. Before reshaping, save a backup ref such as `prepare-gorgeous-pr-backup-<YYYYMMDD>-<HHMMSS>`. A safe, non-interactive way to re-partition is to soft-reset to the base (`git reset --soft <base>`), unstage, and re-commit the code change in logical units by path or crafted patch. Keep `PR-DESCRIPTION.md` out of every product commit. Verify the product tree against the backup as described below. Only ever reshape the branch's own not-yet-shared commits; never rewrite commits already on the base.

## 3. Normalize all prior PR-description history — mandatory

- This step is not part of the optional factoring offer. Perform it whether the user accepted or declined product-commit reshaping, and whether the branch has zero, one, or many existing `PR-DESCRIPTION.md` changes.

- Before rewriting, preserve all previously read description text as context and create a backup ref if step 2 did not already create one. Rewrite only `base..HEAD`: remove the `PR-DESCRIPTION.md` change from every commit that touched it, drop a commit that becomes empty because it contained only that file, and retain every non-description change from a mixed commit in the same logical place. Do this regardless of who authored the old commit. Preserve product commit messages, ordering, and authorship wherever possible; if a mixed commit used the special notes identity, assign its surviving product change to the repository's normal human identity so reviewers cannot mistake product code for notes.

- After normalization, `PR-DESCRIPTION.md` must be absent from every product commit and from the working tree. Verify `git diff <backup> HEAD -- . ':(exclude)PR-DESCRIPTION.md'` is empty. If it is not empty, stop, identify the changed product paths, and do not claim success. Never trade product-tree fidelity for a tidy notes history.

## 4. Write PR-DESCRIPTION.md and commit it as the special author

- The PR definition lives in `PR-DESCRIPTION.md` at the repo root, and is committed by ONE special "notes" author — a deliberate, separate identity, distinct from the code commits:

  ```
  NAME  = Elon Presley
  EMAIL = dmitry.korolev+elon-presley@gmail.com
  ```

- Use the previously read description versions only as a baseline or hint. Take a fresh look at the normalized commits and actual product diff from `base` to `HEAD`, augment accurate prior prose with material behavior now present in the code, replace inaccurate claims, and discard stale or misleading text. Do not rely on where an old notes commit appeared because commits may have been reordered.

- Write `PR-DESCRIPTION.md` as a fresh look at the actual branch changes, using this prompt as the shape of the thinking:

  > take a fresh look at our code changes
  >
  > what's the BLUF / TLDR of the biggest goal of the changes, a few sub-goals you'd highlight, and some implementation details?
  >
  > what was changed/added/removed, and in what components? also separate by language is appropriate and also separate by large code component (py API vs. Rust vs. golang, SQL, etc)

  Prefer this reader-facing shape, modeled on PR #1411:

  ```markdown
  ## Summary
  - [BLUF / TLDR: the biggest goal of the change, in plain terms.]
  - [A second important outcome or sub-goal.]
  - [A third outcome, boundary, or explicit non-goal when useful.]

  ## What This Changes
  [One to three short paragraphs explaining what the code now does differently, in broad strokes first, then a little detail. Write this as an explanation of behavior, not a file-by-file changelog.]

  ## Implementation Details
  - [Concrete mechanism, design choice, or important path through the code.]
  - [Another implementation detail, grouped by component/language when useful.]
  - [Tests or prompt/schema/docs updates may be mentioned here only when they clarify what changed; do not add a separate testing plan.]
  ```

  Write it as a synthesized explanation of what the code does, not as a checklist copied from `git diff`. Start with the user-facing/product or system behavior, then explain how the branch achieves it. Use component/language grouping inside `Implementation Details` when the PR spans large areas such as Python API, Rust, Go, SQL, JavaScript/TypeScript, docs, or infrastructure. Use exactly the three level-two headings shown above, in that order, with no other headings. Do not include a `Testing`, `Test plan`, or `Validation` section, verification commands, expected results, or task-list checkboxes.

  It must follow cleanly from the commit history and the actual code changes: no contradictions, no surprises. A reader should be able to map every claim back to the commits.

- Commit **only** `PR-DESCRIPTION.md` (notes, never code) as a new final commit. Force-add it when the target repository deliberately gitignores local PR drafting notes. Set both author AND committer to the special identity above; setting only `git -c user.name` and `user.email` is insufficient when the environment overrides committer identity, so set `GIT_AUTHOR_NAME`, `GIT_AUTHOR_EMAIL`, `GIT_COMMITTER_NAME`, and `GIT_COMMITTER_EMAIL` explicitly. Use the repository's commit-message convention and do not add any attribution trailer.

## 5. Verify the exact postcondition and report

- Audit every commit in `base..HEAD` again by its actual changed paths. Success requires exactly one commit that changes `PR-DESCRIPTION.md`; that commit must equal `HEAD`, change no other path, and have both author and committer exactly `Elon Presley <dmitry.korolev+elon-presley@gmail.com>`. The file at `HEAD` must contain exactly the three allowed level-two headings in order and none of the forbidden testing, validation, expected-results, verification-command, or checkbox material. The product-tree diff against the backup, excluding `PR-DESCRIPTION.md`, must remain empty.

- If any check fails, stop and report the exact failed invariant, relevant commit SHA, author/committer, or unexpected path. Never describe a merely attempted normalization as successful.

- On success, print the final commit list for `base..HEAD`, state that the product tree is unchanged, show `PR-DESCRIPTION.md`, and identify the unique final Elon Presley notes commit. Remind the user that nothing was pushed and opening the PR remains a separate later step. Write the same report to `tmp/prepare-gorgeous-pr.md`.

## Safety and scope

- Local only. Never push, never open or update a pull request — opening the PR is explicitly a later step. Normalization is mandatory but touches only the branch's own unshared commits, always behind a backup ref and always preserving the product tree. The one intentional identity twist is the special author and committer on the unique final `PR-DESCRIPTION.md` commit; everywhere else, do not add attribution trailers. Any scratch you write goes under the gitignored `tmp/`.
