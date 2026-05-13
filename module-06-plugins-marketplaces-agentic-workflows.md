# Module 6: Reuse Across Repos & Teams — Plugins, Marketplaces, Agentic Workflows

> Part of [Designing Scalable, Reusable GitHub Copilot Customizations](README.md).
>
> **What this module is.** A guide to moving customizations beyond a single repo — bundling them as plugins, sharing them through a marketplace, and extending Copilot from the editor into CI with agentic workflows. For install commands and YAML schemas, see the [official Copilot plugins docs](https://docs.github.com/copilot) and [`github/awesome-copilot`](https://github.com/github/awesome-copilot).

By Module 5 you can build sophisticated customizations inside one repo. The next problem is distribution: ten teams reinventing the same `db-migration` skill, or one team's well-tuned reviewer agent never reaching the rest of the org. This module is about packaging and reach.

> **A note on platforms.** Plugins and marketplaces are Copilot features — they work for any Copilot user, regardless of where the source code is hosted (GitHub, Azure DevOps, GitLab, Bitbucket). **Org-level instructions** and **agentic workflows** are GitHub platform features. If you're on Azure DevOps or another host, the org-tier section in Module 2 and the workflow section below have explicit fallbacks.

---

## 1. Four ways customizations spread

| Mechanism | Who it reaches | When it loads | Best for |
|-----------|----------------|---------------|----------|
| Repository files (`.github/`, `.vscode/`) | Anyone working in that repo | Per the loading rules from Module 1 | Code that lives with one project |
| Org-level instructions *(GitHub Enterprise/Business)* | Everyone in the org | Every request | Policies, standards, security rules |
| Plugin (installed) | Anyone who installs it | Per the loading rules, like local files | Reusable bundles across repos/teams |
| Agentic workflow *(GitHub Actions)* / CI equivalent | Triggered in CI by events | When the event fires (PR opened, issue labeled) | Work that should happen without an editor |

The first two you already know. This module covers the last two.

---

## 2. Plugins: bundling what already works

A plugin is a folder of customizations — skills, custom agents, prompts, MCP server configs, hooks — packaged so another repo can install them as a unit.

The right time to make one is when the same primitive starts being copy-pasted into multiple repos. Not before. A plugin built before anyone has used the underlying skill in anger usually ships the wrong abstraction.

**A reasonable plugin contains:**

- One coherent capability (a domain, not a grab bag)
- The skills, prompts, and agent definitions that go with it
- An MCP server config if the capability needs one
- A short README explaining what it adds and what it assumes

**A plugin that will cause trouble:**

- Bundles unrelated things ("our team's stuff")
- Includes eager instructions that override every consumer's preferences
- Pins a model or hard-codes paths from the original repo

The loading rules don't change inside a plugin. A skill is still lazy. An instruction file is still eager. If the plugin includes a 200-line instructions file, every consumer pays that token cost on every request — so plugin authors should be even more disciplined about what's eager than repo authors.

---

## 3. Marketplaces: where plugins are found

A marketplace is just a registry. `github/awesome-copilot` is the canonical example — a curated list of plugins, instructions, and skills you can install with a single command.

Marketplaces work for any Copilot user. The CLI installs into your local Copilot configuration, so the source repo can live on GitHub, Azure DevOps, GitLab, anywhere. The plugin doesn't care.

Two design questions matter here:

**Should this live in a marketplace at all?** Org-internal policies usually shouldn't. A plugin describing your company's deployment rules belongs in your org's private registry, not a public one. Public marketplaces are for things that generalize: language idioms, framework conventions, well-known APIs.

**Which marketplace?** The public one for community-shared work, an org-internal one for proprietary practices. Many orgs run both — public for general engineering hygiene, private for the parts that encode internal systems. For teams not on GitHub Enterprise, an internal marketplace is just a Git repo (on any host) that your team agrees on as the source of truth, plus a small install convention.

Versioning matters more than people expect. A skill that worked when it was written can mislead the model a year later if the underlying API changed. Treat plugin versions like dependency versions: pin them, review updates, don't auto-upgrade in production repos.

---

## 4. Agentic workflows: Copilot in CI

Agentic workflows let you define Copilot-driven automations as Markdown files that run as GitHub Actions. The trigger is an event — a PR opened, an issue labeled, a comment posted — and the body describes what the agent should do.

This is a different shape of work from the editor. In the editor, a developer is in the loop, judging each suggestion. In a workflow, the agent runs unattended and its output goes straight to a PR comment, an issue update, or a code change.

**Good fits for an agentic workflow:**

- Triage: when a new issue is filed, label it and add a one-paragraph summary
- First-pass review: when a PR opens, post a checklist of what the author should verify
- Routine maintenance: when a dependency update PR appears, run the changelog through a summarizer

**Poor fits:**

- Anything where a wrong answer has real cost (production deploys, security decisions)
- Anything that needs context the workflow can't see (private design docs, recent Slack threads)
- Things a deterministic script does just as well (linting, formatting)

The judgment is the same one as for any automation: what's the cost of being wrong, and how often will it be wrong? Agentic workflows are good at low-stakes, high-volume work. They're bad at irreversible decisions.

### If you're not on GitHub Actions

The agentic-workflow *file format* is GitHub-specific, but the *pattern* is portable. Any CI system that can react to repo events and run a shell command can do the same job:

| Platform | Equivalent surface |
|---|---|
| Azure DevOps | Pipelines triggered by PR / work-item / build events; the pipeline step shells out to the Copilot CLI or a model API and posts back via the ADO REST API |
| GitLab | CI jobs triggered by merge-request hooks; same shape — call the model, post a note via the GitLab API |
| Bitbucket | Pipelines + webhooks |
| Jenkins, CircleCI, etc. | Job triggered by webhook; same pattern |

The portable design lesson: keep the *prompt* (what the agent should do) and the *trigger* (the CI YAML or workflow file) separate. The prompt is just a Markdown file in your repo or plugin — it works the same whether GitHub Actions, ADO Pipelines, or a cron job invokes it. When you migrate platforms, only the trigger needs rewriting.

A practical pattern for ADO customers: store the prompt as `.github/prompts/triage-issue.prompt.md` (yes, even on ADO — the path is just convention; Copilot in VS Code reads it the same way), and have an ADO Pipeline call the Copilot CLI with that prompt file when a work item is created. The skill, the prompt, and the model logic all stay in the repo where developers can edit them; only the pipeline YAML lives in ADO.

---

## 5. What belongs where

A practical decision table when you have something reusable in hand:

| If the thing is... | Put it... | Because |
|--------------------|-----------|---------|
| A rule that applies to one project | Repo instructions | Stays with the code; doesn't pollute others |
| A rule that applies to every project in the org | Org instructions (GitHub EMU/Business) — or a shared template repo / internal plugin elsewhere | One place to update; everyone gets it |
| A capability used by 3+ repos | Plugin | Versioned, installable, removable; works on any host |
| A capability used by the whole industry | Public marketplace plugin | Others benefit; you get fixes back |
| Work that should happen on a repo event, with no developer present | Agentic workflow (GitHub Actions) — or ADO Pipeline / GitLab CI doing the same thing | The editor isn't the right surface |

The mistake to avoid is starting at "plugin" or "workflow" because they sound like the right level of sophistication. Start at the lowest tier that solves the problem. Promote up only when the same thing is being re-solved.

---

## 6. A worked example: a `migration-toolkit` plugin

A team has built, over six months: a `db-migration` skill, a `/migrate-column` prompt, a custom `migration-reviewer` agent, and an MCP server that connects to a staging database for dry-runs. Three other teams have copy-pasted some of these into their repos.

That's the signal to package.

**What goes into the plugin:**

- The skill, the prompt, the agent definition
- The MCP server config (but not the database credentials — those stay per-repo)
- A README explaining: what migrations this assumes (Postgres + a specific migration tool), what it doesn't handle (multi-region, zero-downtime patterns)

**What stays out:**

- The team's general code-review instructions (those are about the team, not migrations)
- A pinned model choice (let the consumer decide)
- Eager instructions ("always check migrations carefully") — that belongs in consuming repos if they want it

**What might also become an agentic workflow:**

When a PR adds a file matching `migrations/*.sql`, a workflow runs the `migration-reviewer` agent on the diff and posts its analysis as a PR comment. Now the same capability works whether the migration was written in the editor or generated by some other process.

The plugin and the workflow are the same logic in two surfaces. That's usually the sign that a piece of work has matured enough to be packaged.

---

## 7. Common mistakes worth avoiding

- **Packaging too early.** A plugin built from one repo's experience usually encodes one repo's assumptions. Wait for the third copy-paste.
- **Mixing concerns.** "Our team's plugin" with reviewer rules, deployment scripts, and a markdown linter is three plugins pretending to be one.
- **Eager instructions inside plugins.** Every consumer pays the token cost on every request. Push as much as possible into skills and prompts.
- **No versioning discipline.** Plugins drift. APIs change. Pin versions in consuming repos and review updates deliberately.
- **Using agentic workflows for high-stakes decisions.** Unattended agents are good at summarizing and triaging. They're bad at deciding what merges.
- **Duplicating between plugin and workflow.** If the same logic runs in both surfaces, factor the prompt or skill out and have both call it.

---

## What to carry into the next module

So far each module has covered one primitive. Module 7 puts them together: a single realistic codebase where instructions, prompts, skills, custom agents, MCP servers, and subagents all play a role — and where the design choice is which primitive owns which part of the work. We'll also look at when a subagent is the right answer and when it's just a longer way to do something the main agent could do directly.
