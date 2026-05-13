# Glossary

> Part of [Designing Scalable, Reusable GitHub Copilot Customizations](README.md).
>
> Definitions used precisely throughout the course. If a term in any module is unclear, look it up here. Each entry includes the module where it's introduced.

## Core concepts

**Primitive.** One of the customization surfaces Copilot recognizes — instructions, prompts, skills, custom agents, MCP servers, hooks, plugins, agentic workflows, subagents. Each primitive has a distinct loading rule and a distinct design intent. *(Module 1.)*

**Loading rule.** When a primitive's content is added to the model's context. The five rules: *every request*, *on glob match*, *on user invoke*, *when the model decides*, *on event*. Choosing the right primitive is mostly choosing the right loading rule. *(Module 1.)*

**Eager / lazy.** Two ways content reaches the model. **Eager** means loaded on every request (instructions, MCP tool *lists*, skill *descriptions*). **Lazy** means loaded only when needed (skill *bodies*, MCP tool *calls*, prompts, hooks). The discipline of the course is to keep the eager surface small. *(Module 1.)*

**Cost (tokens).** Every line in an eager file is paid on every request. The cost callouts in each module remind you of this so eager surfaces don't quietly bloat. *(Module 1.)*

**Context.** Everything the model sees for a request: the user message, attached files, eager customizations, and any lazy content the model chose to load. The course's central optimization is keeping context small without losing relevance. *(Module 1.)*

## Instructions

**Instruction file.** Markdown file with rules the model should generally follow. Loaded eagerly. Three tiers: personal, repository, organization. *(Module 2.)*

**Personal instruction.** Your own preferences, only seen by you. Highest precedence when rules conflict. *(Module 2.)*

**Repository instruction.** Project-wide rules for everyone working in the repo. Lives at `.github/copilot-instructions.md` (or `AGENTS.md`). *(Module 2.)*

**Organization instruction.** Org-wide policy reaching every member. A managed surface on GitHub Enterprise / Business; emulated elsewhere with shared template repos or plugins. *(Module 2.)*

**Scoped instruction.** Markdown file with an `applyTo` glob. Loaded only when an attached file matches. Lives in `.github/instructions/*.instructions.md`. *(Module 2.)*

**`applyTo` glob.** The file pattern that gates a scoped instruction. Narrow globs keep context lean; `**/*` defeats the point. *(Module 2.)*

**`AGENTS.md`.** A more portable repo-instruction format readable by Copilot, Claude, Gemini, and other agents. Use when the repo is consumed by multiple AI tools. *(Module 2.)*

**Parent-repository discovery.** A VS Code setting (`chat.useCustomizationsInParentRepositories`) that lets scoped instructions in a parent repo apply when you open a sub-folder. Important for monorepos. *(Module 2.)*

## Prompts

**Prompt file.** A `*.prompt.md` file that turns a recurring chat ask into a parameterized, invokable workflow. Fires when the user types its slash command. *(Module 3.)*

**Mode.** A prompt's runtime profile: `ask` (chat-only), `edit` (file edits, no tools), or `agent` (full tool access). Pick the smallest that does the job. *(Module 3.)*

**Tool restriction.** A frontmatter field listing which tools a prompt may call. Used to scope behavior, not for security. *(Module 3.)*

**Argument substitution.** Slots like `${file}` or `${input:name}` that the user fills in at invocation time. Two arguments is the comfort zone; three is the ceiling. *(Module 3.)*

## Skills

**Skill.** An on-demand knowledge bundle. The **description** is eager; the **body** is lazy and only loads if the model decides the description matches the task. *(Module 4.)*

**Trigger description.** The short prose at the top of a `SKILL.md` that the model uses to decide whether to read the body. Phrased as `USE WHEN…` / `DO NOT USE FOR…`. The load-bearing piece of any skill. *(Module 4.)*

**Skill body.** The actual playbook content. Loaded only after the description fires. May reference templates, scripts, or examples in a sibling folder. *(Module 4.)*

**Multi-file skill.** A skill organized as a folder with `SKILL.md` plus templates, scripts, or reference material. Used when the body needs supporting assets. *(Module 4.)*

## Agents, MCP, hooks, subagents

**Custom agent.** A `*.agent.md` file that bundles a system prompt, a tool list, and (optionally) a pinned model into a named persona the user can switch into. Loaded when invoked. *(Module 5.)*

**MCP server.** A tool server registered in `.vscode/mcp.json` that exposes new capabilities (databases, APIs, internal services) to the model. The tool *list* is eager; tool *bodies* run only when called. *(Module 5.)*

**Hook.** A command run at a lifecycle event (`PreToolUse`, `PostToolUse`, post-edit). Deterministic — runs regardless of model intent. The right primitive for hard guarantees. *(Module 5.)*

**Subagent.** A stateless worker the main agent dispatches with a focused task. Returns a structured summary. Right primitive when *exploration* is large but the *answer* is small. *(Module 5.)*

**Probabilistic vs. deterministic.** An instruction is *probabilistic* (the model usually follows it). A hook is *deterministic* (the command always runs). Pick by stakes: instructions for preferences, hooks for guarantees. *(Module 5.)*

## Reuse and CI

**Plugin.** A folder of customizations (skills, agents, prompts, MCP configs, hooks) packaged so other repos can install it as a unit. *(Module 6.)*

**Marketplace.** A registry where plugins are discovered and installed. `github/awesome-copilot` is the canonical public one; orgs commonly run a private one too. *(Module 6.)*

**Agentic workflow.** A Copilot-driven automation defined as a Markdown file that runs as a GitHub Action on a repo event (PR opened, issue labeled, comment posted). The CI-side equivalent of a prompt. *(Module 8.)*

## Governance

**Three-tier model.** The split between *enterprise*, *organization*, and *repository* customizations. Each tier owns rules that genuinely apply at that level — and nothing else. *(Module 9.)*

**Ruleset / branch policy.** Platform mechanism for requiring review on changes to specific files (e.g., `copilot-instructions.md`, `mcp.json`, `SKILL.md`). The minimum protection for files that change agent behavior. *(Module 9.)*

**Audit log streaming.** GitHub Enterprise feature that streams agent-related events (session creation, PR activity, policy changes) to a SIEM. The basis for organizational observability. *(Module 9.)*

**AI manager.** A delegated role (formal on GitHub Enterprise, informal elsewhere) that stewards agents, instructions, and MCP configs at the org level — between platform owners and repo maintainers. *(Module 9.)*

## Operations

**Show Agent Debug Logs.** The VS Code panel that records what was loaded into context for each request. The first place to look when something doesn't fire. *(Appendix.)*

**Agent Customizations editor.** The VS Code view of every customization currently active for your workspace, with source and loading rule. Useful for the inverse question: *what's running that I didn't intend?* *(Appendix.)*

**Golden prompt.** A representative question with a written-down "what does a correct answer look like?" Used as a regression check after any customization change. Ten to twenty is enough for team-scale work. *(Appendix.)*

**4,000-character limit.** The cap on how much of a repo's instruction file Copilot code review reads on a PR. Anything past it is silently ignored. *(Module 2 and Appendix.)*
