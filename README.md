# beautiful-skills — a source repo of standalone coding-agent skills

This repository is the **authoring home** for a small family of self-contained, AI-Assisted-Coding skills. Each skill lives in its own directory under `.skills/` and is complete on its own — every rule it relies on is written into its `SKILL.md`, because a skill is deployed by copying its directory, by itself, into a target repository.

## Install with `scsh`

These skills are meant to be installed with **`scsh`** (Scoped Skills Helper — see https://github.com/dkorolev/scsh for details). One command drops the whole family into any repository:

```sh
scsh installskills https://github.com/dkorolev/beautiful-skills
```

`scsh installskills` does all the heavy lifting in the **target** repo: it copies each skill into your `.skills/`, wires the per-harness discovery symlinks (`.claude/skills`, `.codex/skills`, `.cursor/skills`, `.opencode/skills`, `.agents/skills` all pointing at `../.skills`), gitignores `tmp/` so skill scratch never dirties your tree, and merges this repo's `.scsh.yml` entries into your own. That is why this repository ships **no** symlinks of its own — scsh sets those up where they belong, in the consumer. It does keep a one-line `tmp/` `.gitignore` for local sanity (so its own scratch and any in-repo `scsh run`s stay clean); that file is authoring infra and is not installed into consumers.

## The skills

Each skill sits under its own profile in `.scsh.yml`, so a bare `scsh run` is a no-op (it just lists the profiles) and you address one by name. These are interactive, human-in-the-loop skills: the natural way to use one is to have your coding agent invoke it (for example `/big-beautiful-build`), rather than to batch it non-interactively.

- **big-beautiful-build** — one-shot feature factory. Asks once for the full feature description, then delivers working code, a runnable demo, a README, and passing tests, filling every gap with a documented assumption rather than another question.

- **fast-beautiful-forward** — rebases your branch's local commits onto the freshest main of a real remote upstream, so a pull request opened later is a clean fast-forward. Resolves only the conflicts it is certain about and asks about the rest one at a time, and never pushes or opens the PR.

- **code-beautiful-review** — runs the scsh `code-review` reviewer fleet over the branch against local `main`/`master` (no fetch required), then turns the scattered findings into one per-reviewer summary table and clusters important findings separately from stylistic comments.

- **the-beautiful-loop** — loops after `code-beautiful-review`: fixes every important cluster, commits, re-runs `prepare-beautiful-pr` and `code-beautiful-review` until the fleet passes a strict score bar (all routes succeeded, only excellent/good, mean ≥ 4.5). Never pushes or opens the PR.

- **prepare-beautiful-pr** — run after a feature is built to get the branch PR-ready: confirms a clean, non-main branch stacked on main (pointing you at `fast-beautiful-forward` otherwise), offers to factor oversized or mixed commits into focused ones while keeping the code tree byte-identical, ensures `PR-DESCRIPTION.md` is the unique last commit, then writes or updates it using a BLUF `Summary`, `What This Changes`, and `Implementation Details` shape (no separate test plan) and commits it as the special notes author. Never pushes or opens the PR.

- **send-beautiful-pr** — after `prepare-beautiful-pr`: audits commit authorship, drops the local Elon Presley notes commit from what gets pushed, strips `Co-authored-by` trailers (with explicit user approval when needed), pushes the branch for the first time, and opens the PR with `PR-DESCRIPTION.md` as the body.

Each skill declares a `result` report that it writes under the gitignored `tmp/` (`tmp/<skill>.md`).

## Working on the skills

`PRINCIPLES.md` is the **canonical specification** for this family — the conventions every skill must follow. It is an author's reference only: it is **not** shipped (a target repo gets the skill directories alone), and a skill must never refer to it. The duplication between `PRINCIPLES.md` and each self-contained `SKILL.md` is deliberate and essential — it is the deployment mechanism, not a smell.

## Conventions in this repo

- All Markdown files use **one long line per paragraph** — no hard-wrapping within a paragraph.

- Standard ASCII wherever possible; the em dash is the one allowed exception.

- `tmp/` always means the gitignored `tmp/` of whatever repo a skill runs in (scsh gitignores it in the consumer), never the system temp dir; skill scratch and reports go there.
