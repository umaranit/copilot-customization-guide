# Module 8: Agentic Workflows — Copilot in CI

> Part of [Designing Scalable, Reusable GitHub Copilot Customizations](README.md).
>
> **Prerequisites.** [Module 1](module-01-customization-primitives.md) (loading rules), [Module 3](module-03-prompts-as-reusable-workflows.md) (prompts — the editor analogue), [Module 6](module-06-plugins-and-marketplaces.md) (how customizations get distributed).
>
> **What this module is.** A guide to the CI-side surface for Copilot: agentic workflows on GitHub Actions, and the equivalent patterns on Azure DevOps, GitLab, Bitbucket, and Jenkins. For YAML schemas and trigger event names, see the [official Copilot docs](https://docs.github.com/copilot) and the [GitHub Actions docs](https://docs.github.com/actions).

The first seven modules treated Copilot as something a developer interacts with in the editor. This module covers the other surface: Copilot running unattended in CI, reacting to repo events, posting back comments and PRs.

It's a different shape of work — and getting the boundaries right matters more than getting the YAML right.

> **Cost.** Triggered on event. Every run consumes runner minutes and model requests. Cheap individually; expensive when fired indiscriminately. The discipline at this layer is choosing triggers well.

---

## 1. The shape of work that fits CI

In the editor, a developer is in the loop, judging each suggestion. In a workflow, the agent runs unattended and its output goes straight to a PR comment, an issue update, or a code change. That difference reshapes what a workflow should and shouldn't do.

**Good fits for an agentic workflow:**

- **Triage.** When a new issue is filed, label it and add a one-paragraph summary.
- **First-pass review.** When a PR opens, post a checklist of what the author should verify.
- **Routine maintenance.** When a dependency-update PR appears, run the changelog through a summarizer.
- **Linking.** When a PR mentions an issue, post a comment confirming the linked issue actually describes what the diff does.

**Poor fits:**

- **High-stakes decisions.** Production deploys, security approvals, anything irreversible without human review.
- **Tasks needing context the workflow can't see.** Private design docs, recent Slack threads, an incident retro that hasn't been written down yet.
- **Things a deterministic script does just as well.** Linting, formatting, dependency bumps. A workflow that calls a model to do what a `pre-commit` hook would do is paying tokens for nothing.

The judgment is the same one as for any automation: *what's the cost of being wrong, and how often will it be wrong?* Agentic workflows are good at low-stakes, high-volume work. They're bad at irreversible decisions.

---

## 2. The relationship to prompts and skills

An agentic workflow is, in effect, a **CI-triggered prompt**. The body of the workflow is the same kind of content as a `.prompt.md` file from Module 3 — instructions for the agent, an output shape, a list of allowed tools. The only differences are the trigger (a CI event instead of a slash command) and the audience (an automation surface instead of a human).

That means **the same content can run in both surfaces** if you keep them apart cleanly:

- The *prompt or skill* — what the agent should do — lives in the repo as Markdown.
- The *trigger* — when it should run — lives in `.github/workflows/*.yml` (or your platform's equivalent).

Authors who blur this line end up with logic duplicated across the editor and CI, drifting apart. Authors who keep them separate get one source of truth that runs in both places.

*Pattern.* A `triage-issue` skill explains how to label and summarize a new issue. The editor invokes it via a `/triage` prompt that points at the skill. CI invokes it via a workflow that fires on `issues.opened` and runs the same prompt against the new issue's body. The skill is the substance; the prompt and the workflow are two ways to fire it.

---

## 3. Three workflow patterns that earn their keep

Not every team needs all three, but if you're going to run agentic workflows at all, these three are usually the first to add.

### Issue triage on `issues.opened`

When a new issue is filed, an agent reads its body, applies appropriate labels, and posts a one-paragraph summary as a comment. Saves manual triage work and gives downstream automations (label-based routing, SLAs) something to act on.

What earns its keep: **the summary**, not the labels. Labels are easy to get wrong in subtle ways and hard to undo at scale. Summaries are low-stakes — humans read them and ignore them if they're off.

### First-pass PR review on `pull_request.opened`

When a PR opens, an agent reads the diff and posts a checklist comment: "Did you add tests? Does the diff actually do what the linked issue asks for? Are there migration files that should be reviewed manually?" Not a verdict — a *prompt for the human reviewer* to focus on the right things.

What earns its keep: **the checklist shape, not the answers**. An agent that says "this PR looks good" is a liability. An agent that says "here are five things to verify" makes review faster without claiming authority it doesn't have.

### Diff explanation on `pull_request.synchronize`

When a PR is updated, the agent re-reads the diff and updates its prior explanation. Especially useful for big PRs that evolve over a week — the explanation stays current without the author having to update the description by hand.

What earns its keep: **incrementality**. The agent doesn't re-explain everything; it amends what's changed. If your platform doesn't support amending comments cheaply, this pattern is more friction than it's worth.

---

## 4. Off GitHub Actions: the same pattern on other platforms

The agentic-workflow *file format* is GitHub-specific, but the *pattern* is portable. Any CI system that can react to repo events and run a shell command can do the same job:

| Platform | Equivalent surface |
|---|---|
| Azure DevOps | Pipelines triggered by PR / work-item / build events; the pipeline step shells out to the Copilot CLI or a model API and posts back via the ADO REST API |
| GitLab | CI jobs triggered by merge-request hooks; same shape — call the model, post a note via the GitLab API |
| Bitbucket | Pipelines + webhooks |
| Jenkins, CircleCI, etc. | Job triggered by webhook; same pattern |

The portable design lesson: **keep the prompt and the trigger separate**. The prompt is just a Markdown file in your repo or plugin — it works the same whether GitHub Actions, ADO Pipelines, or a cron job invokes it. When you migrate platforms, only the trigger needs rewriting.

A practical pattern for ADO customers: store the prompt as `.github/prompts/triage-issue.prompt.md` (yes, even on ADO — the path is just convention; Copilot in VS Code reads it the same way), and have an ADO Pipeline call the Copilot CLI with that prompt file when a work item is created. The skill, the prompt, and the model logic all stay in the repo where developers can edit them; only the pipeline YAML lives in ADO.

---

## 5. Decision: editor, CI, both, or neither?

Once you have a recurring task, the question is *where it should run*. A small decision table:

| If the task is… | Run it… | Because |
|---|---|---|
| Initiated by a developer when they choose | **Editor (prompt or skill)** | The human is in the loop and judging each step |
| Initiated by a repo event (issue, PR, comment) | **CI (agentic workflow)** | No developer is present; the trigger is a webhook |
| Both — same logic, two triggers | **Author once as a skill or prompt; invoke from both** | Avoid duplicated content drifting apart |
| Deterministic and rule-based | **Neither — use a script, lint, or hook** | A model isn't needed; tokens for nothing |
| Irreversible or security-sensitive | **Editor only, with a human reviewer** | Unattended runs are wrong about edge cases |

The mistake to avoid is reaching for an agentic workflow because it sounds modern. The right reason to add one is: *the work has to happen on a repo event, the developer isn't there to do it, and being wrong is cheap.*

---

## 6. Common mistakes worth avoiding

- **Workflows for work that already has a deterministic answer.** A model-driven "lint" workflow is a paid version of `eslint --fix`. Use the cheaper tool.
- **Workflows that auto-merge or auto-deploy.** Even a 99%-correct agent is wrong about one in a hundred. Apply that to deploys and you'll find out which one.
- **Verbose comments on every PR.** A workflow that posts 500 words of analysis on every change quickly becomes noise that reviewers learn to scroll past. Aim for short, scannable, actionable.
- **Drift between the editor prompt and the CI workflow.** If the same logic exists in two places, it will diverge. Author once; invoke from both.
- **No spending limit on the workflow.** A misconfigured trigger can fire 200 times in an afternoon. Budget like any other CI job — Module 9 covers the cost-governance side.
- **Treating workflow output as authoritative.** It's a draft. Reviewers should still review; triage agents should still be sanity-checked. Workflows assist; they don't decide.

---

## 7. What to carry into the next module

- An agentic workflow is a **CI-triggered prompt**. Same content as Module 3, different surface.
- The right shape of work for a workflow is **low-stakes, high-volume, event-driven**: triage, first-pass review, routine summaries.
- **Author the substance once** (as a prompt or a skill), then invoke it from the editor *and* from CI. Don't duplicate.
- The next module — Module 9 — zooms out to the organization: how to keep all of this (editor customizations, plugins, agentic workflows) consistent, safe, auditable, and cost-bounded as it scales across teams.
