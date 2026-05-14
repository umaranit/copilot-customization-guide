# Module 1: Customization Primitives & Loading Rules

> Part of [Designing Scalable, Reusable GitHub Copilot Customizations](README.md).
>
> **Prerequisites.** [Module 0](module-00-orientation.md) — orientation and reading paths. No prior customization work assumed.
>
> **What this module is.** A simple way to think about Copilot's customization options so you can pick the right one for each job. For setup steps, file paths, and frontmatter fields, see the [VS Code customization docs](https://code.visualstudio.com/docs/copilot/copilot-customization) and the [GitHub Copilot docs](https://docs.github.com/en/copilot/customizing-copilot/about-customizing-github-copilot-chat-responses).
>
> **If something doesn't fire as you read along.** Jump straight to the [Appendix](appendix-evaluating-debugging.md). It covers the two debug surfaces (Show Agent Debug Logs, Agent Customizations editor) that show what actually loaded for each request. Don't wait until the end of the course.

## 1. Think in tiers, not file types

Copilot has many customization files. Instead of memorizing each one, group them by **when they load into the model's context**. That one question answers most design choices.

There are five tiers, ordered from most expensive to cheapest:

| When it loads | How it's triggered | Cost | Examples |
|---|---|---|---|
| **Every time** | Loads with every request, automatically | High — tokens spent on every request | Repo instructions, matched scoped instructions, MCP tool list, skill *descriptions* |
| **When you ask** | You type `/name` or pick a persona | Zero unless you invoke it | Prompts, custom agents |
| **When the model decides** | The model sees a task that fits | Zero unless the model invokes it | Skill *bodies*, MCP tool calls |
| **When something happens** | An event fires (e.g., after a file edit) | Zero — runs outside the model's context | Hooks |
| **When the agent asks for help** | The agent spawns a helper | Low — only a short summary comes back | Subagents |

**Bottom line:** only the top row costs tokens on every request. Keep it short and let the other tiers carry the weight.

## 2. Picking between primitives that look alike

Most "which file should I use?" questions come down to three pairs. Here's a simple way to decide.

### Skill or instruction?

- **Instruction** — short, applies broadly, worth carrying on every request.
- **Skill** — bigger body of knowledge, only relevant for some tasks.

A skill keeps its body out of context until the model needs it. So if your instruction is getting long, it's probably a skill waiting to happen.

- *Instruction example* — "Use TypeScript strict mode." A one-liner that applies to every file in the repo.
- *Skill example* — "How to add a database migration." A multi-step playbook: create the migration file, write the `up` and `down` SQL, test the rollback locally, update the schema doc. Only relevant when someone is actually adding a migration, but valuable in detail when they are.

### Prompt or skill?

- **Prompt** — *you* want to start it (you'll type `/name`).
- **Skill** — you want the *model* to start it when a matching task shows up.

Same idea, different trigger. If you find yourself always invoking something manually, a prompt is the right fit.

*Example.* `/add-test` generates a test file for whatever source file you pass in — you decide when to run it → **prompt**. A skill called `api-error-mapping` should fire whenever the model is editing controllers or error middleware, without you asking → **skill**.

### Custom agent or instruction?

- **Custom agent** — a named persona the user opts into (e.g., `reviewer`, `dba`).
- **Instruction** — a rule that should hold no matter which agent is active.

If you put a rule inside a custom agent, anyone using the default agent won't see it.

*Example.* "Never edit files; only return findings" only makes sense in pre-PR review → **custom agent** (`reviewer`). "All public API changes need a `CHANGELOG.md` entry" should hold whether someone is authoring, reviewing, or using a custom agent → **instruction**.

## 3. A simple trace: "Add a test for `utils.ts`"

Following one real request through the tiers makes the model concrete. The user attaches `src/lib/utils.ts` and types: *"Add a test for utils.ts."*

1. **Always-on instructions load.** The model learns the repo's language, test framework, and where tests live.
2. **Scoped instructions** check their `applyTo` globs. Files that match load; the rest are skipped.
3. **Custom agent** — if the user picked one, its persona and tool limits apply. Otherwise, the default behavior continues.
4. **Prompt** — only fires if the user typed a slash command. Free text doesn't trigger prompts.
5. **Skill descriptions** are already loaded. The model checks if any match the task. If not, no skill body is read.
6. **MCP tools** are listed but not called unless the model decides to.
7. **Subagent** — optional. The model can spawn a helper to explore the codebase and return a one-paragraph summary, keeping the search out of the main thread.
8. **The model acts.** It reads `utils.ts`, then creates `utils.test.ts`. Any hooks configured for file-creation events run their side effects at this point.

Notice how little context was actually spent: a small always-on file, the tool list, the skill descriptions, and maybe one summary from a helper. Everything else stayed off until it was needed. That's the whole game.

## 4. Common mistakes worth avoiding

These are easy to slip into and easy to fix once you spot them.

- **Putting everything in repo instructions.** It loads on every request, even ones it doesn't apply to. Keep it short and move the rest into skills or scoped instructions.
- **Skill descriptions that explain the body.** A description should say *when to use* the skill, not what's inside it. Phrases like "USE WHEN…" and "DO NOT USE FOR…" help the model decide.
- **Using `applyTo: "**/*"` on a scoped instruction.** That's the same as a repo instruction, just harder to find. If it really applies everywhere, put it in repo instructions.
- **Putting team rules inside a custom agent.** Anyone using the default agent won't see them. Rules that always apply belong in instructions.
- **Typing `/skillName` over and over.** That's a prompt, not a skill. If you're the one triggering it every time, give it a slash command.

## 5. The decision cheat sheet

This is the canonical reference for the rest of the course. Each later module opens with a one- or two-row slice of it — the row(s) that module covers in depth — and links back here for the full picture.

| If you want… | Reach for | Loading rule | Cost | Example |
|---|---|---|---|---|
| A rule that should hold across every request | **Repo / personal / org instruction** | Every request | High (eager) | "Use TypeScript strict mode in all new files." |
| A rule that only matters for certain files | **Scoped instruction** (`applyTo` glob) | On glob match | Medium (eager when matched) | `applyTo: "**/*.test.ts"` → "Use Vitest, AAA structure, mock the DB." |
| A workflow *you* will run repeatedly via slash command | **Prompt** | On user invoke | Zero unless invoked | `/add-test` — generates a test file for the attached source file. |
| A body of knowledge the *model* should reach for when relevant | **Skill** | Description always; body when model decides | Low (description only) until used | `db-migration` skill — fires when the model is writing a migration; explains up/down/rollback. |
| A persona with its own tools and defaults | **Custom agent** | On user invoke | Zero unless invoked | `reviewer` agent — read-only tools, "never edit; output `file:line, severity, rule, fix`." |
| A capability the model otherwise can't reach (DB, API, CLI) | **MCP server** | Tool list always; tool call when model decides | Low (tool list only) until called | Postgres MCP server exposing `query` and `describe_table` against the dev DB. |
| A guarantee that must always run, regardless of model intent | **Hook** | On lifecycle event | Zero (runs outside model context) | Post-edit hook runs Prettier on every `.ts` file the agent writes. |
| A wide read where the answer is small | **Subagent** | On main agent's request | Low (only the summary returns) | `Explore` subagent — searches the repo for "where is auth configured?" and returns a one-paragraph summary. |
| A bundle of the above shared across repos | **Plugin** | Same as the contained primitives | Same as the contained primitives | `payments-standards` plugin — ships instructions, a `/add-endpoint` prompt, and a `pci-checklist` skill. |
| Work that should run on a repo event with no developer present | **Agentic workflow** (CI) | On CI event | Per-run runner + model cost | GitHub Action that triages new issues by labeling, summarizing, and assigning on `issues.opened`. |

Keep this table in mind as you read. Every later module is, in effect, *one row of this table, in depth*.

## 6. What to carry into the rest of the course

- Think in **tiers**, not file types. The tier tells you the cost and the activator.
- When two primitives feel similar, ask **who pulls the trigger** (you, the model, an event, or an agent) and **how strict the rule needs to be** (preference or guarantee).
- The next modules go one level deep on each row of the cheat sheet: instructions (Module 2), prompts (Module 3), skills (Module 4), custom agents + MCP + hooks + subagents (Module 5), plugins and marketplaces (Module 6), composition on a real codebase (Module 7), agentic workflows in CI (Module 8), governance at scale (Module 9). The [Appendix](appendix-evaluating-debugging.md) is the operating manual you can flip to at any point.
