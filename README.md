# Scalable & Reusable Agent Design in GitHub Copilot

A self-paced tutorial for working developers who know VS Code and Git, but are new to customizing GitHub Copilot. Eight modules plus one appendix, examples only, no exercises. Aligned with the current GitHub Copilot and VS Code customization taxonomy (instructions, prompts, skills, custom agents, MCP, hooks, plugins, agentic workflows) and the [GitHub Well-Architected guidance on governing agents](https://wellarchitected.github.com/library/governance/recommendations/governing-agents/).

## Scope

For setup steps, file paths, frontmatter fields, and CLI commands, the official sources are your best reference:

- [VS Code: Customize AI](https://code.visualstudio.com/docs/copilot/copilot-customization)
- [GitHub Copilot: Customizing responses](https://docs.github.com/en/copilot/customizing-copilot/about-customizing-github-copilot-chat-responses)
- [`github/awesome-copilot`](https://github.com/github/awesome-copilot) — examples of every primitive

This course sits alongside those references and focuses on **design choices**: which primitive to pick for which job, how loading rules affect context cost, and how the pieces compose into a setup you can reuse across repos and teams. Every module assumes you can look up syntax in the docs and concentrates on the question: *what should you build, and why?*

## Where this works (and where it doesn't)

GitHub Copilot is one product; the GitHub platform is another. Most of this course works for anyone with a Copilot license, regardless of where the source code lives. A few features need GitHub the platform — Enterprise or Business — and one feature class (agentic workflows) needs GitHub Actions specifically.

| Capability | Works with Copilot alone (any repo host: ADO, GitLab, Bitbucket) | Needs GitHub the platform |
|------------|---|---|
| Personal instructions (your own preferences) | Yes | — |
| Repository instructions (`.github/copilot-instructions.md`, `AGENTS.md`, `.github/instructions/*`) | Yes — Copilot reads them from the local working tree regardless of repo host | — |
| Prompts (`*.prompt.md`) | Yes | — |
| Skills (`SKILL.md`) | Yes | — |
| Custom agents (`*.agent.md`) | Yes | — |
| MCP servers (`.vscode/mcp.json`) | Yes — IDE-level | — |
| Hooks | Yes — IDE-level | — |
| Plugins from the public marketplace (`github/awesome-copilot`) | Yes — installed via the Copilot CLI | — |
| Org-wide instructions managed centrally | — | GitHub Enterprise/Business |
| Copilot code review on pull requests | — | GitHub PRs |
| Agentic workflows (Markdown-defined GitHub Actions) | — | GitHub Actions |

**For Azure DevOps / GitLab / Bitbucket users:** everything in Modules 1–5 works the same way. In Module 2, the "organization tier" needs a workaround: a shared template repo or a plugin distributed through your internal marketplace. In Module 6, agentic workflows have direct equivalents — Azure DevOps Pipelines, GitLab CI, Jenkins — that can shell out to the Copilot CLI or a model API for the same outcomes; the pattern is portable even if the YAML isn't. In Module 8, the enterprise-tier policies, audit log streaming, and ruleset enforcement become CI checks, branch policies, and shared-standards repos in your own platform.

The platform-specific bits are called out inline in each module.

---

## Course Outline

### [Module 1 — Customization Primitives & Loading Rules](module-01-customization-primitives.md)
**Learning outcomes**
1. Identify the core Copilot customization primitives — instructions, prompts, skills, custom agents, MCP servers, hooks, plugins — and where each one lives on disk.
2. Explain when each primitive is loaded into the model context (always, on-glob, on-demand, on-invoke, on-event).
3. For each primitive, state a one-line *"choose this when…"* rule.
4. Trace a single user request through the primitives that fire to satisfy it.

**Synopsis.** Module 1 establishes the mental model for everything that follows. Each customization surface is a distinct primitive with a distinct loading rule and a distinct design intent. By the end you can predict, for any user ask, which files contribute to the prompt and in what order — and you have a decision rule for picking the right primitive next time.

### [Module 2 — Instructions That Scale](module-02-instructions-that-scale.md)
**Learning outcomes**
1. Decide between **personal**, **repository**, and **organization** instructions, and understand their precedence.
2. Use repo-wide vs. `applyTo`-scoped instructions without context bloat.
3. Author portable rules in `AGENTS.md` for cross-tool / cross-vendor reuse.
4. Configure parent-repository discovery for monorepos.

**Synopsis.** Instructions are the cheapest, most over-used primitive. This module covers the full three-tier precedence (personal → repository → organization), how `applyTo` globs interact with monorepos, when to use `AGENTS.md` for portability, and how to avoid the "everything is always loaded" trap that silently degrades suggestions.

### [Module 3 — Prompts as Reusable Workflows](module-03-prompts-as-reusable-workflows.md)
**Learning outcomes**
1. Convert a recurring chat ask into a parameterized `.prompt.md`.
2. Use frontmatter (`mode`, `tools`, `description`) to constrain prompt behavior.
3. Compose prompts with instructions and skills without duplication.
4. Scaffold prompts with `/create-prompt`.

**Synopsis.** Prompts are invokable, parameterized chat templates — the smallest unit of reuse above a one-off message. This module covers prompt frontmatter, argument substitution, tool restriction, and how to keep a prompt thin by delegating domain knowledge to skills and project rules to instructions.

### [Module 4 — Skills: Packaged Domain Knowledge](module-04-skills-packaged-domain-knowledge.md)
**Learning outcomes**
1. Distinguish a skill from an instruction or prompt.
2. Author a `SKILL.md` with a precise trigger description (`USE WHEN…` / `DO NOT USE FOR…`).
3. Structure a multi-file skill folder with supporting scripts, templates, and references.
4. Recognize the layout patterns used in `github/awesome-copilot/skills/`.

**Synopsis.** Skills are on-demand knowledge bundles — only the description is in context until the model decides to read the body. This module shows how to write trigger descriptions that fire reliably, how to organize multi-file skills, and when a skill should orchestrate other tools vs. just teach a pattern.

### [Module 5 — Custom Agents, MCP, and Hooks](module-05-custom-agents-mcp-hooks.md)
**Learning outcomes**
1. Author a `.agent.md` (custom agent) that bundles instructions, tools, and a model choice.
2. Configure `.vscode/mcp.json` to add a new tool server.
3. Use **hooks** to enforce deterministic behavior at lifecycle events (`PreToolUse`, `PostToolUse`, post-edit formatters, security guards).
4. Choose between an instruction (preference), a hook (guarantee), and an MCP tool (capability).

**Synopsis.** Custom agents shape *behavior*, MCP adds *capabilities*, hooks enforce *guarantees*. This module clarifies the boundaries: instructions are probabilistic, hooks are deterministic. You will see a minimal MCP server registration, a custom agent that pins a workflow end-to-end, and a hook that blocks a tool call when a policy is violated.

### [Module 6 — Reuse Across Repos & Teams: Plugins, Marketplaces, Agentic Workflows](module-06-plugins-marketplaces-agentic-workflows.md)
**Learning outcomes**
1. Bundle skills, agents, hooks, and MCP servers into an **agent plugin**.
2. Publish to and install from a plugin marketplace (`copilot plugin install <name>@<marketplace>`), using `github/awesome-copilot` as the reference example.
3. Author **agentic workflows** (Markdown-defined GitHub Actions automations) to extend Copilot from the editor into CI.
4. Decide what belongs in a repo, an org, a plugin, or a workflow.

**Synopsis.** Customization stops being personal and starts being organizational here. This module covers the packaging, distribution, and CI-side of Copilot: how plugins bundle multiple primitives behind a single install command, how marketplaces (including the canonical `awesome-copilot` collection) make them discoverable, and how agentic workflows let the same primitives run in GitHub Actions on every PR.

### [Module 7 — Subagents & Composition: Putting It All Together](module-07-subagents-and-composition.md)
**Learning outcomes**
1. Decide when to spawn a subagent vs. handle a task in the main thread.
2. Write subagent prompts that return structured, trustable summaries.
3. Compose instructions, prompts, skills, custom agents, MCP, hooks, and plugins into a repeatable agent design for one realistic codebase.

**Synopsis.** Subagents are stateless, parallelizable workers — ideal for read-heavy exploration and isolated multi-step tasks. This module walks one running example (a `payments-api` repo) end-to-end, showing before/after on context size, latency, and answer quality as each primitive is layered in. By the end you have a template you can copy into a real codebase.

### [Module 8 — Governance & Centralization at Scale](module-08-governance-and-centralization.md)
**Learning outcomes**
1. Decide what belongs at the enterprise, organization, and repository tiers — and what to keep out of every tier.
2. Centralize shared instructions, agents, and MCP configurations without smothering teams in policy.
3. Protect agentic primitive files (`copilot-instructions.md`, `SKILL.md`, `mcp.json`, `.agent.md`) with rulesets, CODEOWNERS, and branch policies.
4. Apply the same review, security, and CI gates to agent-authored code as to human-authored code.
5. Build an observability and audit pipeline (audit log streaming + session monitoring) and configure cost guardrails (spending limits, model allowlists).
6. Translate the enterprise-tier patterns into equivalents for Azure DevOps, GitLab, and Bitbucket teams that don't have GitHub Enterprise.

**Synopsis.** Once dozens of teams are writing skills, agents, and MCP configurations — and once cloud agents are submitting PRs alongside humans — governance becomes the gating concern. This module is anchored in the [GitHub Well-Architected guidance on governing agents](https://wellarchitected.github.com/library/governance/recommendations/governing-agents/) and covers the five concerns (policy, standardization, security & review, observability & audit, cost), the three-tier model in practice, the role split (enterprise / AI managers / repo maintainers), and explicit fallbacks for non-Enterprise teams. It closes with a minimum-viable governance setup you can stand up first and grow from.

### [Appendix — Evaluating & Debugging Customizations](appendix-evaluating-debugging.md)
**Learning outcomes**
1. Inspect what actually loaded with **Show Agent Debug Logs** and the **Agent Customizations editor**.
2. Diagnose common failures: skill descriptions that don't trigger, `applyTo` globs that don't match, conflicting instructions, the 4,000-character limit on Copilot code review.
3. Build a lightweight eval loop (golden prompts + expected outcomes) to catch regressions when primitives change.

**Synopsis.** A short, practical reference for *operating* a customized Copilot setup: how to see what fired, why something didn't, and how to keep your design honest as it grows. Returns to the title's promise — designs that stay scalable and reusable only do so if you can measure them.
