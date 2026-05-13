# Module 5: Custom Agents, MCP, and Hooks

> Part of [Scalable & Reusable Agent Design in GitHub Copilot](README.md).
>
> **What this module is.** A guide to three primitives that change *what* the model can do and *how* it behaves — not what it knows. For frontmatter fields, MCP server configuration, and hook event names, see the [VS Code customization docs](https://code.visualstudio.com/docs/copilot/copilot-customization).

## 1. Three different jobs

The previous modules covered primitives that shape what the model **knows**. This module covers three that shape what the model can **do**:

- **Custom agents** — name a workflow. They bundle a tool set, a system prompt, and (optionally) a pinned model into a persona the user can switch into (e.g., `reviewer`, `dba`).
- **MCP servers** — add new tools. They let the model reach things it otherwise can't: a database, an internal API, a SaaS platform.
- **Hooks** — run side effects at lifecycle events. They make sure something happens (a formatter, a check, a notification) regardless of what the model chose to do.

Each one solves a different problem. Mixing them up is the most common mistake at this layer.

## 2. Custom agents: shaping a persona

A custom agent is the right primitive when a workflow needs **different behavior, a different tool set, or a different model** than your default Copilot setup.

Use one when:

- The workflow has a clear role (reviewer, planner, release manager).
- The model should be restricted to a curated set of tools — not because of security, but to keep the workflow focused.
- You want to pin a specific model so behavior is reproducible across runs.

A custom agent typically holds:

- A **system prompt** describing the role ("you are a strict pre-PR reviewer; never edit files").
- A **tool list** the agent is allowed to use.
- An optional **model** choice.

What it should *not* hold:

- Rules that should apply outside this agent → those belong in instructions.
- One-off workflows the user triggers → those are prompts.

*Example.* A `reviewer` agent has a tight system prompt ("never edit files; output findings as `file:line, severity, rule, fix`"), a read-only tool set (`read_file`, `grep_search`, `get_errors`), and a pinned model. A teammate switches into it before opening a PR and gets the same review every time.

## 3. MCP: extending what the model can reach

Out of the box, Copilot has tools for reading files, searching the workspace, running terminals, and a handful of others. **MCP** is how you add more.

You add an MCP server when the model needs to:

- Talk to a service that isn't already a tool (a database, a ticketing system, a feature-flag platform).
- Use an internal capability (a deployment tool, a custom CLI).
- Pull live data from an API instead of guessing from cached knowledge.

The decision is simple: **does the workflow need a capability the model doesn't have?** If yes, look for an MCP server (or write one). If the capability already exists in some built-in tool, just use that.

Two practical points:

- The **tool list** from every configured MCP server is loaded eagerly. The tool *bodies* run only when called. So adding a server costs you the few lines of metadata for each of its tools, not the whole server.
- An MCP server is a *capability*, not a *behavior*. Adding `postgres-local` doesn't make the model query the database — you still need an instruction, a skill, or a prompt to tell it when querying makes sense.

*Example.* A `payments-api` repo adds an MCP server for its staging database. The model can now run `SELECT` queries when debugging issues. A `db-debug` skill describes when reaching for that tool is appropriate, so the model doesn't do it on every question.

## 4. Hooks: making rules stick

Hooks run a command at a lifecycle event — after a file edit, before a tool call, on session start. Unlike everything else in this course, **hooks are deterministic**: the command runs, regardless of what the model intended.

That makes them the right choice for rules where "the model usually remembers" isn't good enough — a formatter that must run, a check that must pass, a path that must never be written to.

Most repos can live with one or two hooks. They're a tool for specific guarantees, not a general configuration mechanism.

## 5. Choosing between them — and instructions

Once you have all four primitives, the decision becomes: *given this rule or capability, which one fits?*

| What you want | Reach for |
|---|---|
| A preference the model should usually follow | **Instruction** |
| A capability the model can call (database, API, internal service) | **MCP server** |
| A persona with a fixed tool set and pinned model | **Custom agent** |
| A guarantee that runs no matter what the model does | **Hook** |

A worked example: *"every database query in this repo must go through our query helper, not the raw client."*

- As an **instruction**: "Always use `db.query()`, never `client.query()`." → Mostly works. Model sometimes forgets.
- As a **hook**: a pre-tool-use hook that blocks edits introducing `client.query(`. → Works always. Stricter than needed if you trust the team.
- As an **MCP server**: doesn't apply — this is a behavior, not a missing capability.
- As a **custom agent**: doesn't apply unless it's a rule that should only hold inside one workflow (it shouldn't).

For most rules at this stakes level, the right answer is the instruction; for the few where forgetting has a real cost, escalate to a hook.

## 6. Common mistakes worth avoiding

- **Custom agents used as a place to store team rules.** Anyone using the default agent won't see them. Rules that should always hold belong in instructions.
- **MCP servers added "just in case."** Every server's tool list is eager. Add servers when a real workflow needs them, not because they look useful.
- **Treating an MCP server as a behavior.** Adding a tool doesn't tell the model when to use it. Pair it with an instruction or skill that explains the trigger.
- **Reaching for a hook when an instruction would do.** Hooks are deterministic; that's the point. Don't use one for "please use camelCase" — that's an instruction.
- **Pinning a model in a custom agent without reason.** Pinning is useful when reproducibility matters. Otherwise, leave it to the user's default and avoid getting stuck on an old model.

## 7. What to carry into the next module

- These three primitives shape **capability and behavior**, not knowledge. Match the primitive to the job: behavior baseline → instructions, new capability → MCP, named workflow → custom agent, hard guarantee → hook.
- A workflow rarely uses just one. The most useful designs combine them: a custom agent that uses an MCP server, with a hook enforcing the one rule that absolutely must hold.
- The next module covers **plugins and agentic workflows** — how to bundle these primitives for reuse across repos, and how to extend Copilot from the editor into CI.
