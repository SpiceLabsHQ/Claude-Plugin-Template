# Plugin Development Best Practices

Guidance for building high-quality Claude plugins. Skills are the most common component, but this template supports the full plugin feature set.

> **Note:** The Claude plugin format is rapidly evolving. Before acting on specific syntax or field names in this document, use the `claude-code-guide` agent to verify you have the latest information.

## General

- **Each component should have one job.** A skill, agent, or hook that does too much is hard to invoke correctly and harder to maintain.
- **All changes to plugin components are code changes.** Follow the commit type guidance in [CLAUDE.md](../CLAUDE.md#commit-conventions) — using `docs:` or `chore:` for any component edit suppresses the release and leaves the published version stale.

---

## Skills

Skills are prompt templates Claude invokes automatically or on request.

- **Write for automatic invocation first.** Design the description so Claude reaches for the skill naturally — the slash command is a fallback, not the primary path.
- **Prefer depth over breadth.** A focused skill with detailed instructions outperforms a broad skill with vague ones. If you find yourself writing "or" in the description, consider splitting.
- **Inject dynamic context with `!`command``.** Prefixing a shell command with `!` causes it to run at skill invocation time — before the agent receives the prompt — and its output is substituted inline. This is the primary mechanism for giving a skill live, contextual data (current branch, environment state, API response) without the agent having to fetch it:

  ```markdown
  Current git status:
  !`git -C ${CLAUDE_SKILL_DIR} status --short`
  ```

  This is distinct from showing the agent a command to run — the agent never sees the `!`...`` ` line, only its output.

- **Bundle supporting files.** A skill directory can contain secondary markdown docs, reference material, and scripts alongside `SKILL.md`. Keep the main file under 500 lines and extract the rest — Claude reads supporting files on demand via markdown links:

  ```text
  my-skill/
  ├── SKILL.md
  ├── reference.md       ← linked from SKILL.md, loaded on demand
  └── scripts/
      └── helper.py      ← executed by the agent via Bash
  ```

  For scripts the agent should run, show them as regular code — the agent decides when and how to invoke them, and can pass its own arguments. Use `${CLAUDE_SKILL_DIR}` so paths resolve regardless of working directory:

  ```markdown
  To process results, run:
  `python ${CLAUDE_SKILL_DIR}/scripts/helper.py <output-file>`
  ```

  Any script the agent may execute should be covered by `allowed-tools`.

- **Grant scoped tool permissions with `allowed-tools`.** Skills can pre-authorize specific tools for their duration, eliminating approval prompts without permanently expanding permissions. Use permission rule syntax to narrow the grant:

  ```yaml
  allowed-tools: Bash(gh *), Read, Grep, Glob
  ```

  This approves only `gh` subcommands within Bash — not arbitrary shell execution. Include any bundled scripts the agent needs to run.

- **Test before committing.** Use the pre-enabled `skill-creator` skill (`/skill-creator`) to run evals and iterate against representative inputs.

For naming and description rules, see [CLAUDE.md](../CLAUDE.md#naming-and-description).

---

## Agents

Agents are specialized subagents Claude can spin up for a defined scope of work.

- **Scope them narrowly.** An agent for "security review" is more predictable than one for "reviewing everything." The description should state exactly when Claude should delegate to this agent.
- **Prefer agents over complex skills for multi-step work.** If a skill needs to use many tools or run a long sequence of steps, an agent with `context: fork` is often a better fit.

---

## Hooks

Hooks run automatically in response to Claude Code lifecycle events.

- **Use matchers precisely.** A hook on `Write|Edit` runs on every file write — fine for a fast validator, wrong for an expensive process. Use the `if` field to narrow further when needed.
- **Keep hooks fast.** Slow hooks block the session. Offload expensive work using `async: true` or delegate to a background agent.

---

## MCP Servers

MCP servers connect Claude to external tools and services.

- **Use `${CLAUDE_PLUGIN_ROOT}` for paths.** Hardcoded paths break on other machines. Reference binaries and config files relative to the plugin root.
- **Prefix tool names with the service.** `database_query` is unambiguous; `query` will collide. Namespace everything.

---

## LSP Servers

LSP servers add code intelligence for languages Claude doesn't natively support.

- **Check the marketplace first.** Common languages already have LSP plugins. Only ship one if a gap genuinely exists.
- **Document the required binary.** State exactly which language server users must install and how, or the plugin silently does nothing.

---

## Commands

Commands use the same syntax and support the same frontmatter features as skills, with two differences:

- **Commands cannot bundle supporting files.** A command is a single file; it cannot reference sibling scripts or documentation the way a skill directory can.
- **Invocation paths differ.** Commands appear in autocomplete as `/PLUGIN_NAME:COMMAND_NAME`. Skills that are user-invocable use `/skill-name` syntax but do not support autocomplete at time of writing.

Choose commands when the autocomplete discoverability matters to your users and the skill doesn't need bundled files. Choose skills when you need to bundle scripts, reference docs, or inject dynamic context via `!`command``.

---

## User Configuration

User config fields prompt users for input at enable time (API endpoints, tokens, flags).

- **Mark sensitive fields correctly.** Set `"sensitive": true` for tokens and passwords (stored in the system keychain); `"sensitive": false` for everything else (stored in settings).
- **Describe the expected format.** Users shouldn't have to guess whether a field wants a full URL, a bare hostname, or a 40-character token.

---

## Plugin Manifest

The `plugin.json` manifest declares all components and drives marketplace distribution.

- **Bump `version` before distributing changes.** Without a version increment, cached installs won't update.
- **Keep the manifest minimal.** Only override default component paths when your repo structure requires it — the defaults handle the common case.
