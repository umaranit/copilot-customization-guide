# Designing Scalable, Reusable GitHub Copilot Customizations

A self-paced tutorial for working developers who know VS Code and Git, but are new to customizing GitHub Copilot. Ten modules plus one appendix and a glossary, examples only, no exercises. Aligned with the current GitHub Copilot and VS Code customization taxonomy (instructions, prompts, skills, custom agents, MCP, hooks, subagents, plugins, agentic workflows) and the [GitHub Well-Architected guidance on governing agents](https://wellarchitected.github.com/library/governance/recommendations/governing-agents/).

**Start here:** [Module 0 — Orientation](module-00-orientation.md). It explains who the course is for, what it focuses on (and what it doesn't), where each capability does or doesn't work, and how to navigate the modules depending on your role.

---

## Course outline

The course is grouped into four arcs. Each module assumes the previous one; jump-in points for specific roles are listed in [Module 0](module-00-orientation.md#5-suggested-reading-paths).

### Foundation

#### [Module 0 — Orientation](module-00-orientation.md)
Who the course is for, scope, where it works (Copilot vs. GitHub-the-platform), reading paths, conventions used throughout.

#### [Module 1 — Customization Primitives & Loading Rules](module-01-customization-primitives.md)
**Learning outcomes**
1. Identify the core Copilot customization primitives — instructions, prompts, skills, custom agents, MCP servers, hooks, subagents, plugins, agentic workflows — and where each one lives on disk.
2. Explain when each primitive is loaded into the model context (always, on-glob, on-demand, on-invoke, on-event).
3. For each primitive, state a one-line *"choose this when…"* rule.
4. Trace a single user request through the primitives that fire to satisfy it.

**Synopsis.** Establishes the mental model for everything that follows. Each customization surface is a distinct primitive with a distinct loading rule and a distinct design intent. Closes with the *decision cheat sheet* that subsequent modules re-show, annotated, as each row gets covered in depth.

### Primitives, one at a time

#### [Module 2 — Instructions That Scale](module-02-instructions-that-scale.md)
**Learning outcomes**
1. Decide between **personal**, **repository**, and **organization** instructions, and understand their precedence.
2. Use repo-wide vs. `applyTo`-scoped instructions without context bloat.
3. Author portable rules in `AGENTS.md` for cross-tool / cross-vendor reuse.
4. Configure parent-repository discovery for monorepos.

**Synopsis.** Instructions are the cheapest, most over-used primitive. Three-tier precedence (personal → repository → organization), how `applyTo` globs interact with monorepos, when to use `AGENTS.md` for portability, and how to avoid the "everything is always loaded" trap.

#### [Module 3 — Prompts as Reusable Workflows](module-03-prompts-as-reusable-workflows.md)
**Learning outcomes**
1. Convert a recurring chat ask into a parameterized `.prompt.md`.
2. Use frontmatter (`mode`, `tools`, `description`) to constrain prompt behavior.
3. Compose prompts with instructions and skills without duplication.
4. Scaffold prompts with `/create-prompt`.

**Synopsis.** Prompts are invokable, parameterized chat templates — the smallest unit of reuse above a one-off message. Frontmatter, argument substitution, tool restriction, and how to keep a prompt thin by delegating domain knowledge to skills and project rules to instructions.

#### [Module 4 — Skills: Packaged Domain Knowledge](module-04-skills-packaged-domain-knowledge.md)
**Learning outcomes**
1. Distinguish a skill from an instruction or prompt.
2. Author a `SKILL.md` with a precise trigger description (`USE WHEN…` / `DO NOT USE FOR…`).
3. Structure a multi-file skill folder with supporting scripts, templates, and references.
4. Recognize the layout patterns used in `github/awesome-copilot/skills/`.

**Synopsis.** Skills are on-demand knowledge bundles — only the description is in context until the model decides to read the body. How to write trigger descriptions that fire reliably, organize multi-file skills, and decide when a skill should orchestrate other tools vs. just teach a pattern.

#### [Module 5 — Custom Agents, MCP, Hooks, and Subagents](module-05-custom-agents-mcp-hooks-subagents.md)
**Learning outcomes**
1. Author a `.agent.md` (custom agent) that bundles instructions, tools, and a model choice.
2. Configure `.vscode/mcp.json` to add a new tool server.
3. Use **hooks** to enforce deterministic behavior at lifecycle events.
4. Decide when to spawn a **subagent** vs. handle a task in the main thread.
5. Choose between an instruction (preference), a hook (guarantee), an MCP tool (capability), a custom agent (persona), and a subagent (read-heavy worker).

**Synopsis.** Custom agents shape *behavior*, MCP adds *capabilities*, hooks enforce *guarantees*, subagents absorb *exploration*. This module clarifies the boundaries between probabilistic and deterministic primitives, and between primitives that change what the model knows vs. what it can do. Closes the primitive set; later modules are about packaging, composing, extending into CI, and governing.

### Reuse and synthesis

#### [Module 6 — Reuse Across Repos & Teams: Plugins and Marketplaces](module-06-plugins-and-marketplaces.md)
**Learning outcomes**
1. Bundle skills, agents, hooks, and MCP servers into an **agent plugin**.
2. Publish to and install from a plugin marketplace, using `github/awesome-copilot` as the reference example.
3. Decide what belongs in a repo, an org, or a plugin.

**Synopsis.** Customization stops being personal and starts being organizational here. How plugins bundle multiple primitives behind a single install command, how marketplaces (including the canonical `awesome-copilot` collection) make them discoverable, and the versioning discipline that keeps shared customizations honest. The CI side is its own module — see Module 8.

#### [Module 7 — Composition Walkthrough: Primitives on `payments-api`](module-07-composition-walkthrough.md)
**Learning outcomes**
1. Apply the primitives from Modules 2–5 in combination on one realistic codebase.
2. Diagnose, for any specific failure, which primitive the fix belongs in.
3. Take a copy-able template into your own repo.

**Synopsis.** A full walkthrough of a `payments-api` repo from "Copilot installed, nothing customized" to a setup where instructions, prompts, skills, custom agents, MCP, hooks, and subagents each own a piece of the work — without ending up with so much customization that every request burns through the context window.

### Beyond the editor

#### [Module 8 — Agentic Workflows: Copilot in CI](module-08-agentic-workflows-in-ci.md)
**Learning outcomes**
1. Author **agentic workflows** (Markdown-defined GitHub Actions automations) for triage, first-pass review, and routine maintenance.
2. Translate the same pattern to Azure DevOps Pipelines, GitLab CI, or any webhook-driven CI.
3. Author the substance once (as a prompt or skill) and invoke it from both the editor and CI.

**Synopsis.** The other surface for Copilot: running unattended in CI on repo events. Which shapes of work fit (low-stakes, high-volume), which don't (irreversible decisions), and how to keep the editor and CI surfaces from drifting apart.

#### [Module 9 — Governance & Centralization at Scale](module-09-governance-and-centralization.md)
**Learning outcomes**
1. Decide what belongs at the enterprise, organization, and repository tiers.
2. Centralize shared instructions, agents, and MCP configurations without smothering teams in policy.
3. Protect agentic primitive files (`copilot-instructions.md`, `SKILL.md`, `mcp.json`, `.agent.md`) with rulesets, CODEOWNERS, and branch policies.
4. Apply the same review, security, and CI gates to agent-authored code as to human-authored code.
5. Build an observability and audit pipeline (audit log streaming + session monitoring) and configure cost guardrails.
6. Translate the enterprise-tier patterns into equivalents for Azure DevOps, GitLab, and Bitbucket teams.

**Synopsis.** Anchored in the [GitHub Well-Architected guidance on governing agents](https://wellarchitected.github.com/library/governance/recommendations/governing-agents/). Covers the five concerns (policy, standardization, security & review, observability & audit, cost), the three-tier model in practice, the role split (enterprise / AI managers / repo maintainers), and explicit fallbacks for non-Enterprise teams. Closes with a minimum-viable governance setup.

### Operating the system

#### [Appendix — Evaluating & Debugging Customizations](appendix-evaluating-debugging.md)
**Learning outcomes**
1. Inspect what actually loaded with **Show Agent Debug Logs** and the **Agent Customizations editor**.
2. Diagnose common failures: skill descriptions that don't trigger, `applyTo` globs that don't match, conflicting instructions, the 4,000-character limit on Copilot code review.
3. Build a lightweight eval loop (golden prompts + expected outcomes) to catch regressions when primitives change.

**Synopsis.** A short, practical reference for *operating* a customized Copilot setup: how to see what fired, why something didn't, and how to keep your design honest as it grows. **Flip to it the moment any primitive seems not to fire** — it's the operating manual you can use at any point in the course.

#### [Glossary](glossary.md)
Every term used precisely throughout the course, with the module where it's introduced. Use it as a self-test after each module.
