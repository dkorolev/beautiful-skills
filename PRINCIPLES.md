# PRINCIPLES.md

This is the **canonical specification** — the **principles** every skill in this family must follow — kept here as a reference for the author. It is not a runtime artifact and does **not** ship with the skills: each skill is deployed by copying its own directory into a target repository, on its own, without this file. Every rule a skill relies on must therefore live inside that skill itself — so the duplication between this file and the skills is deliberate and essential, the deployment mechanism rather than a smell. A skill must never refer to this file or assume it is present.

All Markdown files, including this one, should not break lines. Use one long line per paragraph.

In Markdown, when a list (numbered or bulleted) is made of multiple multi-line records — items that each run to more than one line when rendered (a full sentence or more), or that carry their own sub-content such as a nested list or code block — leave one blank line between consecutive items. Lists of short, single-phrase items may stay tight, with no blank lines.

Avoid non-ASCII characters wherever possible. The one allowed exception is the em dash (`—`), which we all use these days. Otherwise stick to standard ASCII: for instance, write an ellipsis as three periods (`...`), never as a single-glyph Unicode ellipsis.

When you edit a skill, check it against the rules below to confirm it still honors them and does not contradict the others. The sections here are the source of truth; each skill is a self-contained restatement of the parts it needs. If a rule changes, update every skill that carries it.

## 1. Skill definition

- Each skill is an AI-Assisted Coding Agent Skill defined by a `SKILL.md` with valid YAML frontmatter containing at least `name` and `description`.

- `name` MUST exactly match the skill's directory name.

- The `description` MUST state what the skill does and when it should trigger, so the harness invokes it for the right request.

## 2. The `tmp/` convention

- Throughout this family, **`tmp/` means the gitignored `tmp/` subdirectory of the repository the skill is working in** — never the operating system's temp directory (which we call "the system temp dir").

- Anything a skill writes **back into the repository** as scratch — intermediate output, logs, generated artifacts that are not part of the deliverable — goes under `tmp/`, so it stays gitignored and is never committed by accident. Respect the repository's root `.gitignore`.

- The deliverable itself (the code, demo, README, and tests the skill produces) is **not** scratch — it lands where it belongs in the repository. The exception to this rule is when the deliverable of a skill is to be consumed by another skill, and is not meant to be committed — in which case it should go under `tmp/` of the repo as instructed.

Skills should be designed to not conflict with one another, so what a file or a directory is created under `tmp/`, it should follow some convention, such as `{purpose}-{YYYYMMDD}-{HHMMSS}-{random}`, where `random` is perhaps six random lowercase Latin characters. For programmatic chaining of skills, the skill may request to have the output artefact name provided to it from the outside, as the parameter, or as an environmental variable.

Of course, if the skill is designed specifically to produce a particular file under `tmp/` of the repository, this is perfectly acceptable — as long as it is mentioned in the description in the skill that it is not designed to be run concurrently. Even in this case though, it's a good practice to first write a random file (as `tmp/{purpose}-{YYYYMMDD}-{HHMMSS}-{random}.{json|txt|md|yml}`), and then atomically rename it into the desired final output filename.

## 3. Git hygiene and the commit trailer

- A skill commits its work by **adding commits on top of the current branch**. If invoked from the repository's main branch (`main` or `master`), commit straight onto it — skills should not spin up a side branch unless instructed specifically to do so.

- **Never rewrite shared history** (no rebasing or squashing of commits that already exist upstream); only append. Reordering or rewording your own not-yet-shared commits is fine.

- Match the repository's own commit-message conventions — mirror the subject/body style of recent commits.

- Do not add attribution trailers. We do not want `Co-Authored-By` in the repo. This is the house convention. If a repository's own `CONTRIBUTING.md` states a different trailer, follow the repository's.

## 4. Safety boundaries

- Commit only when the work calls for it (or the user asks). **Never push, open a pull request, publish, or take any other outward-facing action without an explicit request.** Those — along with anything destructive or irreversible (deleting the user's data, spending money) — are the universal safety boundaries, and they hold regardless of any other instruction.

- Before a destructive or irreversible step, stop and confirm first.

- Before any step with side effects, confirm with the user first. This has to do with git commits, github pull requests, comments, responses, reactions, task tracker responses, email/slack/DM messages, etc. The skills are encouraged to seek the information they need to complete the task, and the key is that the skills should be **reading** this information freely — but **mutating the state**, with any human-visible side effects, should require approvals. A major exception is local commits, which the skills should do liberally as they need to — including creating local branches for themselves when the skill explicitly instructs this.

## 5. House style

- **Read the repository's own playbook first** — `CONTRIBUTING.md` and any agent/model instruction files (`AGENTS.md`, `CLAUDE.md`, including nested ones) — and hold the work to it. The repository's stated conventions win over these general principles wherever they conflict. Keep in mind that there are many other conventions we need to follow: some repositories have the CONSTITUTION with potential AMENDMENTS and PRECEDENTS, some have the MAXIMS to be followed — be sure to discover them and analyze them.

- Match the surrounding code's naming, comment density, and idiom for whatever language the work is in — write code that reads like it belongs.

- Format before finishing (e.g. `cargo fmt`, `prettier`), keep files free of trailing whitespace, and end every file with a newline.

## 6. Self-check in the authoring repository

These apply only in this authoring repository (`beautiful-skills`). They do **not** ship with any skill — a target repo gets the skill directories alone, installed by `scsh installskills`; the manifest and the self-check stay here.

- **Complete manifest.** The skills live under `.skills/` — one directory per skill, at `.skills/<name>/SKILL.md`. The repo root keeps a `.scsh.yml` whose `skills:` section lists **every** shipped skill — one entry per `.skills/<name>/` directory, keyed by the skill's `name` (matching the directory) — plus the `internal-self-check-beautiful-skills` entry itself. The manifest must be complete: no skill omitted, no listed name without a directory. Each shipped skill entry declares `harness`, `model`, `timeout`, its own `profile: <name>`, and `result: tmp/<name>.md`. The file is **skills-only**: scsh supplies the base image, so do not add `version`, `project`, or `image` headers.

- **`internal-self-check-beautiful-skills`.** An authoring-only skill (directory `.skills/internal-self-check-beautiful-skills/`, **never copied** to a target repo) — and the one skill that is **not** a shipped family skill, so sections 1-5 do not bind it; it enforces them on the others. Both its `internal-` name **and** an explicit `autoinstall: false` keep `scsh installskills` from ever shipping it. It carries `profile: internal-self-check` and `result: tmp/internal-self-check-beautiful-skills.json`, so it runs under `scsh run --profile internal-self-check`; a bare `scsh run` is a no-op, since every shipped skill sits under its own profile. Run it after committing. It reads `PRINCIPLES.md`, the manifest, and each shipped skill's `SKILL.md`, and confirms every skill is self-contained — it never refers to `PRINCIPLES.md` — and faithfully restates the rules it relies on. Unlike the shipped skills, it **may** reference `PRINCIPLES.md`, because it never leaves this repo.

- **When you add or remove a skill.** Create or delete the `.skills/<name>/` directory **and** add or remove its entry in `.scsh.yml` in the same change, then run `scsh run --profile internal-self-check` (after committing) to confirm the family still passes self-check.
