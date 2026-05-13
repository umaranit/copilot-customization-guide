# Module 0: Orientation — How to Read This Course

> Part of [Designing Scalable, Reusable GitHub Copilot Customizations](README.md).
>
> **What this module is.** A short orientation: who the course is for, what it focuses on (and what it deliberately doesn't), where each capability does or doesn't work, and how to navigate the modules depending on your role. Read this once, then move to [Module 1](module-01-customization-primitives.md).

## 1. Who this is for

Working developers who already know VS Code and Git, but are new to customizing GitHub Copilot. The course is self-paced — ten modules plus an appendix, examples only, no exercises.

You'll get the most out of it if you've used Copilot Chat for at least a few days, have opened a `.github/` folder before, and have a real repo in mind to apply ideas to as you read.

## 2. What this course focuses on (and doesn't)

**Focus.** *Design choices* — which primitive to pick for which job, how loading rules affect context cost, and how the pieces compose into a setup you can reuse across repos and teams. Every module concentrates on the question: *what should you build, and why?*

**Not the focus.** Setup steps, file paths, frontmatter fields, and CLI commands. Those change frequently and are documented well already. The official sources are your best reference:

- [VS Code: Customize AI](https://code.visualstudio.com/docs/copilot/copilot-customization)
- [GitHub Copilot: Customizing responses](https://docs.github.com/en/copilot/customizing-copilot/about-customizing-github-copilot-chat-responses)
- [`github/awesome-copilot`](https://github.com/github/awesome-copilot) — examples of every primitive

This course sits alongside those references. Every module assumes you can look up syntax in the docs.

## 3. Where this works (and where it doesn't)

GitHub Copilot is one product; the GitHub platform is another. Most of this course works for anyone with a GitHub Copilot license, regardless of where the source code lives. A few features need GitHub the platform, and one feature class (agentic workflows) needs GitHub Actions specifically.

| Capability | Works with Copilot alone (any repo host: ADO, GitLab, Bitbucket) | Needs GitHub the platform |
|------------|---|---|
| Personal instructions | Yes | — |
| Repository instructions (`.github/copilot-instructions.md`, `AGENTS.md`, `.github/instructions/*`) | Yes — Copilot reads them from the local working tree regardless of host | — |
| Prompts (`*.prompt.md`) | Yes | — |
| Skills (`SKILL.md`) | Yes | — |
| Custom agents (`*.agent.md`) | Yes | — |
| MCP servers (`.vscode/mcp.json`) | Yes — IDE-level | — |
| Hooks | Yes — IDE-level | — |
| Plugins from the public marketplace (`github/awesome-copilot`) | Yes — installed via the Copilot CLI | — |
| Org-wide instructions managed centrally | — | GitHub Enterprise / Business |
| Copilot code review on pull requests | — | GitHub PRs |
| Agentic workflows (Markdown-defined GitHub Actions) | — | GitHub Actions |

**For Azure DevOps / GitLab / Bitbucket users:** everything in Modules 1–5 and the capstone in Module 7 works the same way. In Module 2, the "organization tier" needs a workaround (a shared template repo or a plugin distributed through your internal marketplace). In Module 8, agentic workflows have direct equivalents — Azure DevOps Pipelines, GitLab CI, Jenkins — that can shell out to the Copilot CLI or a model API for the same outcomes; the pattern is portable even if the YAML isn't. In Module 9, the enterprise-tier policies, audit log streaming, and ruleset enforcement become CI checks, branch policies, and shared-standards repos in your own platform.

The platform-specific bits are called out inline in each module.

## 4. The shape of the course

Ten modules and one appendix, grouped into four arcs:

| Arc | Modules | Question it answers |
|---|---|---|
| Foundation | 0, 1 | What primitives exist, when do they load, and how do I think about them? |
| Primitives one at a time | 2, 3, 4, 5 | How do I author each primitive well? |
| Reuse and synthesis | 6, 7 | How do I package what works and compose primitives on a real codebase? |
| Beyond the editor | 8, 9 | How do I run agents in CI, and how do I govern all of this at scale? |
| Operating the system | Appendix | How do I tell whether any of this is actually working? |

Modules build on each other. Where a module needs something from an earlier one, it says so at the top. The appendix is a debugging reference — **flip to it the moment any primitive seems not to fire**. Don't wait until you've finished the course.

## 5. Suggested reading paths

- **Building a customized setup for one repo (default path).** Read in order: 0 → 1 → 2 → 3 → 4 → 5 → 7. Skip 6, 8, 9 until you need them. Keep the Appendix open in another tab.
- **Setting up Copilot for a team or org (admin path).** Read 0 → 1 → 9 first, to establish the governance shape. Then come back for 2 → 4 → 6 to understand what teams will actually be authoring.
- **Adding Copilot to CI (automation path).** Read 0 → 1 → 4 → 6 → 8. Modules 2, 3, 5 are useful background but not blocking.
- **Just want the mental model.** Read 0 → 1, then skim the *"common mistakes"* sections in the rest.

## 6. Conventions used throughout

- **Cost callouts.** When a primitive is introduced, you'll see a small box noting whether it loads *every request*, *on glob match*, *on demand*, *on user invoke*, or *on event*. That's the single most important thing about each primitive.
- **Decision tables.** Each module that introduces a primitive ends with a re-shown decision table — the same table grows as primitives accumulate.
- **Common mistakes.** Almost every module has one. They're worth more than the rest of the module on a re-read.
- **What to carry forward.** A short closing section that names the bridge to the next module.

## 7. A glossary, in case you want it

A separate [glossary](glossary.md) collects every term used precisely in the course (primitive, scoped instruction, trigger description, subagent, hook, plugin, agentic workflow, etc.). It's a good self-test after each module: if you can read a term and recall when it loads and when to reach for it, you've got the module.

---

Ready? [Module 1 — Customization Primitives & Loading Rules →](module-01-customization-primitives.md)
