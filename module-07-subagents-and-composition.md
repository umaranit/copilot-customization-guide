# Module 7: Subagents & Composition — Putting It All Together

> Part of [Designing Scalable, Reusable GitHub Copilot Customizations](README.md).
>
> **What this module is.** A walkthrough that puts every primitive together on one realistic codebase. We follow a `payments-api` repo from "Copilot installed, nothing customized" to a setup where instructions, prompts, skills, custom agents, MCP, hooks, and subagents each own a piece of the work. For subagent syntax and tool details, see the [VS Code subagent docs](https://code.visualstudio.com/docs/copilot/copilot-customization).

The first six modules looked at primitives one at a time. The real skill is composition — knowing which primitive owns which part of the work, and when adding another piece actually helps versus when it just adds noise.

---

## 1. The running example

`payments-api` is a Node.js + TypeScript service. It owns customer payment methods and charge processing. Three things make it interesting from a Copilot perspective:

- **Strict conventions.** Every database query goes through one helper. Every external API call has a retry wrapper. PRs that violate either bounce in code review.
- **A migrations problem.** Schema changes are frequent and risky. Bad migrations have caused two outages.
- **A large codebase.** ~80k lines, 200+ files, several years old. New contributors take weeks to find their way around.

A fresh contributor with default Copilot will get plausible code that ignores both conventions, suggests migrations that look reasonable but break in production, and answers "where does X happen?" with confident wrong guesses.

The job of this module is to fix all of that — without ending up with so much customization that every request burns through the context window.

---

## 2. Layer 1 — Instructions: the always-on baseline

Start with what every request should know. These are the things that, if the model forgets, the answer is wrong.

**Repo-wide (`.github/copilot-instructions.md`):**

- The stack: Node 20, TypeScript strict, Postgres, Express
- The two non-negotiable rules: queries through `db.query()`, external calls through `withRetry()`
- Where things live at the top level: `src/`, `migrations/`, `tests/`

**Scoped (`.github/instructions/tests.instructions.md`, `applyTo: "**/*.test.ts"`):**

- Use Jest, not Mocha
- Mock the database with the test helper, not by stubbing `db`
- Group tests by behavior, not by method

That's it. Two files, both small. The temptation here is to keep adding rules. Resist it — every line is paid on every request. If a rule only matters sometimes, it belongs in a skill or a prompt, not an instruction.

**Why not put the migration rules here?** Because migrations are touched by maybe 5% of requests. Pushing the migration playbook into always-on instructions means the other 95% of requests pay for context they don't use. Migrations get a skill instead.

---

## 3. Layer 2 — Skills: knowledge that loads when needed

Two skills earn their keep on this codebase:

**`skills/db-migration/SKILL.md`** — fires when the model is reading or writing files in `migrations/`, or when the user asks about schema changes. Body covers: the migration file naming convention, the up/down pattern, the three things that have caused outages before, the staging-database dry-run procedure.

**`skills/codebase-tour/SKILL.md`** — fires when the user asks "where does X happen" or "how does Y work". Body is a map of the codebase: which directory owns what, which files are entry points, which abstractions are load-bearing vs. incidental.

Both are lazy. The descriptions are eager (a few hundred tokens total), but the bodies — which together are several thousand tokens — only enter context when the model decides they're relevant. That's the whole point of skills: pay the small cost of advertising them, pay the larger cost only when the work needs them.

---

## 4. Layer 3 — Prompts: workflows that should be one command

Three workflows on this codebase repeat enough to deserve prompts:

- **`/add-test`** — given a function, generate a Jest test that mocks the DB correctly. Tools restricted to read + edit. Composes with the test instructions automatically.
- **`/migrate-column`** — given a table and column change, draft an up/down migration following the project's pattern. Composes with the `db-migration` skill.
- **`/explain-here`** — given a file or function, walk through what it does and what it depends on. Read-only.

Each prompt is thin: a few lines describing the task, an argument or two, a tool restriction. None of them duplicate what's in instructions or skills — they invoke them.

The signal that a workflow deserves a prompt was given in Module 3: the third time you find yourself typing the same chat ask, write it down. Two of these started as Slack snippets that the team kept copy-pasting.

---

## 5. Layer 4 — Custom agents: when the persona matters

One custom agent on this codebase: **`reviewer`**.

The reviewer agent's job is to look at a diff and call out problems. It needs different defaults from the general assistant: it should never edit, it should be explicit when something is unclear instead of guessing, and it should follow a checklist (the two non-negotiable rules, plus a few migration-specific checks).

It's a custom agent rather than just a prompt because the persona is sustained — once you're in `reviewer` mode, every follow-up question carries those defaults. A prompt resets after one exchange. Code review is a conversation, not a one-shot.

There is *not* a `migration-author` custom agent, even though it was tempting. The work is too narrow — once a migration is drafted, you're back to normal coding. A prompt (`/migrate-column`) plus a skill is enough; a separate agent persona would just be ceremony.

---

## 6. Layer 5 — MCP: capabilities the model otherwise can't reach

Two MCP servers are configured for this repo:

- **`postgres-staging`** — read-only access to the staging database. Lets the reviewer agent (and the migration prompt) check what tables actually look like, not just what the schema files say.
- **`internal-issue-tracker`** — read access to the team's issue tracker. Lets the model find the original issue when reviewing a PR, so it can check whether the diff actually does what was asked.

Both are scoped narrowly. Neither has write access. The principle from Module 5 holds: every tool inflates the eager tool list, and every tool the model can use is a tool that can be used wrong. Two narrow tools beat ten broad ones.

What's *not* an MCP server: a generic "filesystem" tool (the model already has file access), a "run any SQL" tool against production (no — too dangerous; staging-only is the right scope), or a Slack search tool that nobody actually uses (cut it).

---

## 7. Layer 6 — Hooks: the deterministic backstop

Two hooks, kept light:

- A **post-edit hook** runs Prettier on any TypeScript file the agent writes. Means the agent never has to remember formatting rules — the hook handles it.
- A **pre-tool hook** blocks any attempt to write to `migrations/` outside of a dedicated branch. Gives a clear error message pointing the user at `/migrate-column`.

Both are deterministic guarantees, not advice. That's the line between an instruction and a hook from Module 5: instructions are usually-followed, hooks are always-enforced. The pre-tool hook is there because "never edit migrations directly" was previously a repo instruction, and it was being violated about once a month.

---

## 8. Layer 7 — Subagents: when to spawn one

A subagent is a stateless worker the main agent can dispatch to. It gets a focused task, does it, and returns a structured summary. The main agent's context never sees the subagent's work in detail — only the summary.

The right time to spawn one is when a task would otherwise blow up the main context: reading 30 files to find a pattern, exploring an unfamiliar module, doing a wide search where most results are irrelevant.

**On `payments-api`, three good subagent uses:**

- **"Where is X used?"** When the question requires reading dozens of call sites. The subagent reports back: "12 usages, here are the 3 that look relevant." The main agent doesn't need the other 9 files in its context.
- **"Does this PR break any tests?"** The subagent runs the relevant tests, reads any failures, and returns a one-paragraph verdict.
- **"Find similar code."** The subagent does a semantic + grep sweep and returns a ranked list. The main agent then decides which one to look at in detail.

**Bad uses on this codebase:**

- "Explain this function." The main agent can read one file. Spawning a subagent adds latency for no benefit.
- "Make these three edits." Edits should stay in the main thread — they need to be visible to the user as they happen.
- "Pick a name for this variable." Far too small. Subagent overhead exceeds the work.

The rule of thumb: spawn a subagent when the *exploration* is large but the *answer* is small. A subagent that returns a 2,000-token report defeats its own purpose — at that point you've just shifted the context bloat, not eliminated it.

---

## 9. The composition payoff

Here's what changes for someone working on `payments-api` after all of this is in place:

| Task | Before | After |
|---|---|---|
| "Add a test for `chargeCustomer`" | Generic Jest stub, mocks `db` directly, fails the team's review | `/add-test` produces a test using the project's mock helper, follows the test instructions, passes review |
| "Why is `withRetry` wrapping every call to Stripe?" | Plausible-sounding wrong answer | `codebase-tour` skill fires; answer references the right file and the original incident the wrapper was added for |
| "Change the `email` column to `varchar(320)`" | Migration that misses the index, causes downtime | `/migrate-column` + `db-migration` skill produces an up/down pair with the index, pre-tool hook prevents direct edits |
| Reviewing a PR | Generic "looks good", misses convention violations | `reviewer` agent checks both non-negotiables and the migration rules; flags the issue tracker if the diff doesn't match the linked issue |
| "Where do we handle webhook signature verification?" | "It's probably in `src/middleware/`…" (wrong) | Subagent searches; returns one file path with a one-line explanation |

None of this required heavy customization on every layer. Two instruction files, two skills, three prompts, one custom agent, two MCP servers, two hooks. Total ongoing token cost is dominated by the two instruction files and the skill descriptions — everything else is paid only when relevant.

That's the goal: a setup that scales with the codebase's complexity, not with the number of features it has.

---

## 10. A template for your own repo

Working backwards from a real codebase, ask in this order:

1. **What's wrong without any customization?** Be specific. "The model writes the wrong SQL pattern." "It can't find anything." "It doesn't know our test helper."
2. **For each problem, what's the loading rule that fits?**
   - Is this needed on *every* request? → Instruction.
   - Is this needed when *certain files* are touched? → Scoped instruction.
   - Is this needed only when the *user asks something specific*? → Skill.
   - Is this a *workflow* the user runs repeatedly? → Prompt.
   - Is this a *persona* — sustained behavior across a conversation? → Custom agent.
   - Is this a *capability* the model otherwise lacks? → MCP.
   - Is this a *guarantee* that must always hold? → Hook.
   - Is this a *wide read* where the answer is small? → Subagent.
3. **What's the smallest thing that would solve the problem?** Build that. Don't pre-emptively add layers.
4. **Measure.** Did the problem actually go away? Are answers slower, faster, smaller, larger? The appendix covers how to evaluate this honestly.

The whole course can be summarized in one sentence: *match the loading rule to the cost, and pick the smallest primitive that solves the problem.* The rest is taste.

---

## What to carry into the next module

Module 7 was a single-codebase view: how all the primitives compose for one team. Module 8 zooms out to the organization: how dozens of teams keep their customizations consistent, safe, auditable, and cost-bounded without becoming a bottleneck. The appendix that follows both modules is the operational complement — how to tell, day to day, whether any of this is actually working.
