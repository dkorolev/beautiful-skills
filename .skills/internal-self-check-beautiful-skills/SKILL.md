---
name: internal-self-check-beautiful-skills
description: Authoring-only meta-check for the beautiful-skills repo. Reads PRINCIPLES.md, the .scsh.yml manifest, and every shipped skill's SKILL.md, then verifies the manifest is complete and each skill is self-contained and faithfully restates the principles in PRINCIPLES.md. It is not a shipped family skill and is never copied to a target repo (its internal- name and its autoinstall: false both exclude it); run it with `scsh run --profile internal-self-check`.
---

# Self-Check: beautiful-skills

You audit the **skill family** in this authoring repository against `PRINCIPLES.md`. You build nothing and ship nothing — you check that every skill still conforms to the principles `PRINCIPLES.md` lays out and could be copied, alone, into a target repository and still work. You only report; a human fixes what you flag.

You are authoring-only: you live in this repo, are never copied to a target repo (your `internal-` name and your `autoinstall: false` both keep `scsh installskills` from shipping you), and you **may** read `PRINCIPLES.md` directly — the shipped skills may not. sections 1-5 describe what the *shipped skills* must be; they do not bind you. Because your directory is `.skills/internal-self-check-beautiful-skills/`, you never audit yourself.

## What you read

- `PRINCIPLES.md` — the canonical spec: the two Markdown rules and the non-ASCII rule at the top, then sections 1-6.

- `.scsh.yml` — the manifest at the repo root.

- `README.md` and `CONTRIBUTING.md` — the repo's own docs.

- Every shipped skill's `SKILL.md` the manifest lists — that is, every `.skills/<name>/` directory except your own.

## What you check

**Manifest (section 6).** `.scsh.yml` is skills-only — no `version`, `project`, or `image` headers. Its `skills:` section has exactly one entry per `.skills/<name>/` directory (each keyed by a `name` that matches its directory), plus your own `internal-self-check-beautiful-skills` entry. No skill directory is missing; no listed name lacks a directory. Each shipped skill entry declares `harness`, `model`, `timeout`, its own `profile: <name>`, and `result: tmp/<name>.md`; your entry carries the same `harness`/`model`/`timeout`, plus `profile: internal-self-check`, `result: tmp/internal-self-check-beautiful-skills.json`, and `autoinstall: false`.

**Each shipped skill (the top rules and sections 1-5).** For every listed skill except yourself:

- **Section 1** — valid YAML frontmatter with a `name` that matches the directory and a `description` that states what the skill does and when it triggers.

- **Section 2** — the `tmp/` convention: scratch and any declared `result` go under the repository's gitignored `tmp/`, never the system temp dir, while the real deliverable lands where it belongs.

- **Section 3 and section 4** — git hygiene and safety are described wherever the skill acts: it commits onto the current branch and never rewrites shared history; it never adds a `Co-Authored-By` or other attribution trailer; and it never pushes, opens a pull request, publishes, or takes any other outward-facing or destructive action without an explicit user request.

- **Section 5** — it reads the repository's own playbook first and matches house style.

- **Self-contained** — it restates, in its own words, every rule it depends on, and it **never refers to `PRINCIPLES.md`** or assumes that file is present; it would still work copied alone into a target repo. Links to a target repo's own `CONTRIBUTING.md`, `README.md`, or `.skills/README.md` are by design — they resolve in the consumer — and are **not** findings.

- **No contradiction** — nothing in the skill conflicts with the principles or with another skill's stated purpose. Minor overlap between skills is fine.

**Markdown and ASCII conventions.** Every Markdown file the check touches — `PRINCIPLES.md`, `README.md`, `CONTRIBUTING.md`, and each `SKILL.md` — obeys the rules at the top of `PRINCIPLES.md`: one long line per paragraph, with no paragraph hard-wrapped across several lines; one blank line between the items of a multi-line list, while a list of short single-phrase items may stay tight; and standard ASCII only, the em dash (`—`) being the sole allowed non-ASCII character. Flag a Unicode ellipsis (it should be spelled `...`), a middle dot, a no-break space, a smart quote, or any other stray non-ASCII glyph. A non-ASCII glyph shown inside backticks purely as an example of what to avoid is not a finding. A hard-wrapped paragraph, packed multi-line list items, or a stray non-ASCII character is a finding.

## Output

Write `tmp/internal-self-check-beautiful-skills.json` — a single JSON object of this shape:

```ts
type Status = "pass" | "fail";

interface SelfCheck {
  result: { status: Status; problems_found: number };  // problems_found MUST equal problems.length
  problems: Problem[];
}

interface Problem {
  skill: string;       // the skill's name, or ".scsh.yml" for a manifest-level problem
  principle: string;   // the rule at stake, e.g. "section 2", "section 4 safety", "section 6 manifest", "ASCII", "Markdown"
  description: string;  // what does not conform
  suggestion: string;  // how to bring it back into line (advice only — never applied)
}
```

`status` is `fail` when `problems` is non-empty and `pass` otherwise. When everything conforms, emit `problems: []` and `status: "pass"`.

## Tone

Terse and direct, findings only. You report; a human resolves.
