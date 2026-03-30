# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## Repository purpose

This repo is a Claude plugin. It contains one or more skills under `skills/` that are distributed as a Claude Code plugin and as standalone Claude Desktop skill packages.

## Commit conventions

This repo uses [Conventional Commits](https://www.conventionalcommits.org/). All commits must follow the format:

```
<type>(<optional scope>): <description>
```

Common types and their effect on versioning:

| Type | When to use | Release |
|------|-------------|---------|
| `feat` | New skill or meaningful new behavior | minor |
| `fix` | Corrects unintended skill behavior | patch |
| `refactor` | Restructures skill content without changing behavior | patch |
| `docs` | README, CLAUDE.md, comments only — **never for `skills/` changes** | none |
| `chore` | Dependency updates, config changes — **never for `skills/` changes** | none |
| `ci` | Changes to GitHub Actions workflows | none |

A `BREAKING CHANGE:` footer on any commit triggers a major release.

Any change to files under `skills/` is a code change and must use a release-triggering type (`feat`, `fix`, `refactor`, etc.). Using `docs` or `chore` for skill edits will suppress the release and leave the published version stale.

## Skills

> **Note:** The Claude skill format is rapidly evolving. Before adding or significantly editing a skill file, use the `claude-code-guide` agent to check for the latest supported frontmatter fields, syntax changes, and best practices.

To create a new skill, use the `skill-creator` skill — it is pre-enabled in this repo and handles scaffolding, iteration, and evaluation:

```
/skill-creator
```

Each skill lives in `skills/<skill-name>/SKILL.md`. When adding or editing a skill:

- Frontmatter must include a `description:` field
- Keep `SKILL.md` under 500 lines; extract supporting content to separate files in the same directory
- The skill name (directory name) should be lowercase and hyphenated

### Naming and description

The `name` and `description` fields are the primary signals the agent uses when deciding whether to automatically invoke a skill. Get these wrong and the skill will be ignored or misapplied.

**Name** — use a verb-noun pair that describes the action, not the domain:
- Good: `review-pr`, `generate-changelog`, `triage-issue`
- Avoid: `pr-helper`, `changelog`, `issues` (too generic or noun-only)

**Description** — write it from the agent's perspective: *when should I reach for this skill?* It should describe the trigger condition, not the implementation. Keep it under 250 characters — descriptions are truncated in the skill listing, and all skills in a plugin share a fixed context budget, so longer descriptions crowd out others.
- Good: `"Use when the user asks to review a pull request for correctness, style, or security issues"`
- Avoid: `"Reviews pull requests"` (describes what, not when)
- Avoid: `"A helpful skill for PR review tasks"` (vague, no trigger signal)

**Disambiguating similar skills** — if two skills could match the same request, prefer scoping with the `paths` frontmatter field over trying to word descriptions differently. A skill with `paths: "src/ui/**"` will only be considered when working with matching files, which is more reliable than description-level differentiation alone.

**Other frontmatter that affects invocation:**
- `paths` — Claude only considers the skill when working with files matching the glob
- `disable-model-invocation: true` — Claude never auto-invokes this skill; it must be called explicitly with `/skill-name`. Use this for skills that should only run at user request.
- `user-invocable: false` — hides the skill from the `/` menu so users never see or call it directly. Use this for skills designed to be exclusively called by other skills or agents rather than by humans.

## CI

The CI pipeline (`.github/workflows/ci.yml`) runs on every push and PR to `main`:

1. **Validate** — checks `plugin.json` structure and each skill's `SKILL.md`
2. **Release** — runs only on `main` after validation passes; semantic-release determines the version bump, updates `plugin.json`, generates `CHANGELOG.md`, packages skills as `.zip` files, and creates a GitHub Release
