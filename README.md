# beautiful-skills — a source repo of standalone coding-agent skills

This repository is the **authoring home** for a small family of self-contained, AI-Assisted-Coding skills. Each skill lives in its own directory under `.skills/` and is complete on its own — every rule it relies on is written into its `SKILL.md`, because a skill is deployed by copying its directory, by itself, into a target repository.

## Install — two commands, machine-wide

```sh
cargo install scsh
```

```sh
scsh installskills --global \
  https://github.com/dkorolev/beautiful-skills \
  https://github.com/dkorolev/code-review-skills
```

That is the whole setup. `scsh installskills --global` (scsh 1.25+) installs this family and the five-reviewer fleet machine-wide: the skills land under `~/.scsh/.skills/`, their profiles merge into the **global manifest** `~/.scsh/.scsh.yml`, and every skill is symlinked into the user-level skills dir of each coding agent already present on the machine (`~/.claude/skills`, `~/.cursor/skills`, `~/.codex/skills`, ...). From then on the skills work in **any** git repository — `/code-gorgeous-review` runs `scsh run code-gorgeous-review`, whose gorgeous-specific name ignores an older repo-local `code-review` profile and therefore resolves from the global manifest — and no `.scsh.yml` or `.skills/` ever lands in the reviewed repo. The agent-facing wrapper itself uses the separate `code-gorgeous-review-driver` profile, so it never joins or recursively launches the 15-reviewer fleet.

To upgrade an existing global install, first upgrade `scsh`, then refresh both the skill files and their global manifest blocks:

```sh
cargo install scsh
scsh updateskills --global \
  https://github.com/dkorolev/beautiful-skills \
  https://github.com/dkorolev/code-review-skills
```

**Why `-gorgeous-`?** The scsh binary bundles an older snapshot of this family under the original `-beautiful-` names (that is what a bare `scsh installskills --global`, with no URLs, installs). This repository's shipped skills are TEMPORARILY named `-gorgeous-` (`big-beautiful-build` keeps its name) so the fresh, from-source family installs and runs alongside the bundled one without any name collisions — in the global manifest, in the agents' skills dirs, and in repos that already carry `-beautiful-` installs.

## Uninstall — it is all just files

A global install is three kinds of plain files, so removal is plain `rm` — there is no uninstaller:

- **Skill bodies** are real directories under `~/.scsh/.skills/<name>/`.

- **Agent entries** are per-skill symlinks at `~/.claude/skills/<name>`, `~/.cursor/skills/<name>`, `~/.codex/skills/<name>`, ... pointing into `~/.scsh/.skills/`. Remove the symlink itself with plain `rm` — no `-r`, and no trailing slash (a trailing slash would make `rm` follow the link into the real directory).

- **The global manifest** is `~/.scsh/.scsh.yml`, one block per skill under `skills:`. Delete the skill's block — do not leave an entry whose `~/.scsh/.skills/<name>/` is gone, or `scsh` would resolve the profile and then fail on the missing skill body.

One skill:

```sh
rm ~/.claude/skills/the-gorgeous-loop ~/.cursor/skills/the-gorgeous-loop   # each agent you use
rm -rf ~/.scsh/.skills/the-gorgeous-loop
# ...then delete its block from ~/.scsh/.scsh.yml
```

The whole gorgeous family (the reviewers and `big-beautiful-build` follow the same per-name pattern):

```sh
for a in ~/.claude ~/.cursor ~/.codex ~/.opencode ~/.agents; do rm -f "$a"/skills/*-gorgeous-*; done
rm -rf ~/.scsh/.skills/*-gorgeous-*
# ...then delete their blocks from ~/.scsh/.scsh.yml — or `rm ~/.scsh/.scsh.yml` to drop EVERY globally-installed profile
```

## Per-repo install with `scsh` (optional)

The skills can also be dropped into one repository — for a team that wants them committed and versioned with the code:

```sh
scsh installskills https://github.com/dkorolev/beautiful-skills
```

`scsh installskills` does all the heavy lifting in the **target** repo: it copies each skill into your `.skills/`, wires the per-harness discovery symlinks (`.claude/skills`, `.codex/skills`, `.cursor/skills`, `.opencode/skills`, `.agents/skills` all pointing at `../.skills`), gitignores `tmp/` so skill scratch never dirties your tree, and merges this repo's `.scsh.yml` entries into your own. That is why this repository ships **no** symlinks of its own — scsh sets those up where they belong, in the consumer. It does keep a one-line `tmp/` `.gitignore` for local sanity (so its own scratch and any in-repo `scsh run`s stay clean); that file is authoring infra and is not installed into consumers.

## The skills

Each skill sits under its own profile in `.scsh.yml`, so a bare `scsh run` is a no-op (it just lists the profiles) and you address one by name. These are interactive, human-in-the-loop skills: the natural way to use one is to have your coding agent invoke it (for example `/big-beautiful-build`), rather than to batch it non-interactively.

- **big-beautiful-build** — one-shot feature factory. Asks once for the full feature description, then delivers working code, a runnable demo, a README, and passing tests, filling every gap with a documented assumption rather than another question.

- **fast-gorgeous-forward** — rebases your branch's local commits onto the freshest main of a real remote upstream, so a pull request opened later is a clean fast-forward. Resolves only the conflicts it is certain about and asks about the rest one at a time, and never pushes or opens the PR.

- **code-gorgeous-review** — runs the scsh `code-gorgeous-review` reviewer fleet (resolved from the repo's own exact-name profile or the global manifest — nothing installed into the target repo, and a legacy `code-review` profile is ignored) over the branch against local `main`/`master` (no fetch required), then turns the scattered findings into one per-reviewer summary table and clusters important findings separately from stylistic comments.

- **the-gorgeous-loop** — loops after `code-gorgeous-review`: fixes every important cluster, commits, re-runs `prepare-gorgeous-pr` and `code-gorgeous-review` until the fleet passes a strict score bar (all routes succeeded, only excellent/good, mean ≥ 4.5). Reviews and fixes run against a base pinned once at the start — the loop never gates an iteration on sitting atop the freshest main; rebasing (`fast-gorgeous-forward`) is a single step after it converges. Never pushes or opens the PR.

- **prepare-gorgeous-pr** — run after a feature is built to get the branch PR-ready: confirms a clean, non-main branch stacked on main (pointing you at `fast-gorgeous-forward` otherwise), offers to factor oversized or mixed commits into focused ones while keeping the code tree byte-identical, ensures `PR-DESCRIPTION.md` is the unique last commit, then writes or updates it using a BLUF `Summary`, `What This Changes`, and `Implementation Details` shape (no separate test plan) and commits it as the special notes author. Never pushes or opens the PR.

- **send-gorgeous-pr** — after `prepare-gorgeous-pr`: audits commit authorship, drops the local Elon Presley notes commit from what gets pushed, strips `Co-authored-by` trailers (with explicit user approval when needed), pushes the branch for the first time, and opens the PR with `PR-DESCRIPTION.md` as the body.

- **kickoff-gorgeous-pipeline** — the whole chain in one invocation: `fast-gorgeous-forward` first (the only interactive stage — upstream questions and rebase conflicts are settled with the user up front, and the local base branch is fast-forwarded to the same tip), then unattended to the end: `prepare-gorgeous-pr` with its commit-reshaping offer suppressed, one `code-gorgeous-review` round, and `the-gorgeous-loop` until the fleet passes the bar or a cluster is stuck three rounds. Never pushes — `send-gorgeous-pr` stays a separate human step.

Each skill declares a `result` report that it writes under the gitignored `tmp/` (`tmp/<skill>.md`).

## Working on the skills

`PRINCIPLES.md` is the **canonical specification** for this family — the conventions every skill must follow. It is an author's reference only: it is **not** shipped (a target repo gets the skill directories alone), and a skill must never refer to it. The duplication between `PRINCIPLES.md` and each self-contained `SKILL.md` is deliberate and essential — it is the deployment mechanism, not a smell.

## Conventions in this repo

- All Markdown files use **one long line per paragraph** — no hard-wrapping within a paragraph.

- Standard ASCII wherever possible; the em dash is the one allowed exception.

- `tmp/` always means the gitignored `tmp/` of whatever repo a skill runs in (scsh gitignores it in the consumer), never the system temp dir; skill scratch and reports go there.
