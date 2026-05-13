# Module 8: Governance & Centralization at Scale

> Part of [Designing Scalable, Reusable GitHub Copilot Customizations](README.md).
>
> **What this module is.** A guide to the questions that come up once Copilot customizations stop being one team's problem and become an organization's problem: who controls what, how do you keep it consistent, how do you audit it, how do you keep costs predictable, and how do you do all that without becoming a bottleneck. Anchored in the [GitHub Well-Architected guidance on governing agents](https://wellarchitected.github.com/library/governance/recommendations/governing-agents/), with explicit fallbacks for teams not on GitHub Enterprise.

The first seven modules treated customizations as a developer problem: which primitive to pick, how to compose them, how to debug them. This module treats them as an organizational problem. Once dozens of teams are writing their own skills, agents, and MCP configurations — and once cloud agents are submitting PRs alongside humans — the failure modes shift. A misconfigured shared agent, an unreviewed MCP server, or an unmanaged spend limit can affect many repositories at once.

The good news: most of the design discipline from the earlier modules carries over. The new question is *who decides what at which level*, not *what does a good skill look like*.

---

## 1. Where governance actually lives

There are five concerns that need an owner once you scale beyond one team:

| Concern | The question it answers |
|---|---|
| **Policy** | What models, agents, and tools is the organization allowed to use at all? |
| **Standardization** | What baseline rules and shared customizations does every team start from? |
| **Security & review** | What reviews and guardrails apply equally to human and agent-authored changes? |
| **Observability & audit** | Who can see what an agent did, how long are those records kept, and what do you alert on? |
| **Cost** | Who pays, what are the limits, and how do you attribute spend? |

Each of these can be handled at the enterprise level (one place, applies everywhere), the organization level (per business unit or product), or the repository level (per project). Picking the right level matters more than picking the right rule. Push everything up and you bottleneck every team; push everything down and you have no consistency.

The Well-Architected guidance summarizes this as: *minimize enterprise-level baseline, empower organizations to adapt.* Set the non-negotiable controls centrally — typically audit log streaming, model allowlists, MCP allowlists — and let organizations layer their own configuration on top.

---

## 2. The three-tier model in practice

| Tier | What belongs here | Why |
|---|---|---|
| **Enterprise** *(GitHub Enterprise/Business)* | Audit log streaming, model allowlist, third-party-agent toggle, AI manager role assignments, spending limits, MCP allowlist policy | These are compliance and safety floors. Every org inherits them; no org can opt out. |
| **Organization** | Org-wide custom instructions (security, compliance baseline), enterprise custom agents (`.github-private`), MCP registry curation, code-review automation policy, runner defaults | This is where business units adapt the baseline to their domain — payments has different needs from internal tooling. |
| **Repository** | Repo `.github/copilot-instructions.md`, scoped `.github/instructions/*.instructions.md`, prompts, skills, MCP server configs, `copilot-setup-steps.yml`, hooks | This is where the real effectiveness gains happen — language, framework, codebase-specific. |

The trap at every tier is putting too much in. Org-wide instructions that try to cover every possible case become 3,000 tokens that load on every request, costing money and degrading answers. Enterprise policy that mandates a specific MCP server for everyone slows down the team that has a better one. As Module 1 said: every line in an eager file is paid on every request — that math gets brutal at organizational scale.

**For non-Enterprise teams (ADO, GitLab, Bitbucket):** the enterprise tier doesn't exist as a managed surface. The equivalent shape is:

- A central **shared-standards repo** (or template repo) that publishes the baseline instructions, agents, and MCP configs.
- A **plugin** (Module 6) distributed via your internal marketplace for portable bundles.
- **CI checks** in your pipeline platform that enforce things the GitHub-enterprise rulesets would otherwise enforce — for example, refusing to merge PRs that modify `.github/copilot-instructions.md` without a designated reviewer.

You lose the ability to mandate centrally-enforced policy with a flip of a switch. You gain it back through repo conventions, plugin pinning, and CI gates. Slower to roll out, but the governance shape is the same.

---

## 3. Standardization without smothering

Once a team owns "shared standards", the next failure mode is over-shipping: 200 lines of org-wide instructions covering everything from variable naming to deployment procedures. Most of it never gets read by the model in any useful way, and all of it costs tokens.

**A useful shared-standards layer contains:**

- **Non-negotiable security rules** — never log credentials, never disable security headers, follow OWASP secure-coding practices.
- **Compliance language** the company is required to state (e.g., licenses, attribution rules).
- **One or two coding hygiene rules** that genuinely apply everywhere (e.g., "always generate tests for new public functions").

**It does not contain:**

- Style preferences (those belong in personal instructions).
- Stack-specific rules ("we use TypeScript with strict mode") — those belong in repo or team-template instructions.
- Long-form policy documents — those belong in skills, where the body is loaded only when relevant.
- Anything Copilot can't actually act on ("communicate with kindness").

The pattern: **rules live at the level closest to the code they apply to.** Push a rule up only when it genuinely applies to every repo in the org.

---

## 4. Centrally-managed customizations: shared libraries, enterprise agents, plugins

Three mechanisms for sharing customizations across teams without copy-paste:

**Shared instruction library.** A repository (call it `org-copilot-standards`) that publishes starter `copilot-instructions.md`, common scoped instructions for popular stacks, and reference skills. Teams reference or copy what they need. Versioned, reviewed, owned by a small team. Works for any host (GitHub, ADO, GitLab) — it's just a repo.

**Enterprise custom agents** *(GitHub Enterprise specific)*. Agents defined in a designated `.github-private` repository become available across the entire enterprise. Use this for things like a standard "reviewer" agent that encodes the company's review checklist. Day-to-day management can be delegated to AI managers (a custom role) so security-sensitive decisions stay with enterprise owners while the agent definitions stay close to the teams that use them. **Non-Enterprise alternative:** publish the same agent definitions as a plugin in your internal marketplace — same outcome, slightly more friction to install.

**Plugins (from Module 6).** The portable equivalent. A bundled skill + agent + prompt + MCP config that any team can install. Best for capabilities that aren't universal but are reusable — a `migration-toolkit` that only repos with databases need, a `frontend-a11y-checks` plugin that only UI repos need.

Across all three: **test before you broadcast.** A shared agent that ships with a vague description, or an over-eager skill that fires on every database question, will degrade everyone's experience until someone fixes it. The Well-Architected guidance is explicit: roll out enterprise customizations to a sandbox or pilot group first, validate, then expand.

---

## 5. Protecting the customization files themselves

If the customizations *are* the governance, the files defining them have to be protected. An unreviewed change to `.github/copilot-instructions.md`, `SKILL.md`, `mcp.json`, or an `.agent.md` can change agent behavior across many sessions and many repositories silently.

**On GitHub:** apply rulesets to these files so changes require human review. The Well-Architected guidance explicitly calls out:

- `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`
- `.github/copilot-instructions.md`
- `.github/instructions/**/*.instructions.md`
- `SKILL.md`
- `mcp.json` / `.github/copilot/mcp.json`
- The `.github-private` repository for enterprise custom agents
- `copilot-setup-steps.yml` (controls the agent's environment)

These files should be CODEOWNERS-protected and require independent review before merge. Treat them like infrastructure-as-code, because effectively they are.

**On ADO/GitLab/Bitbucket:** the same files exist (under the same paths or equivalents — Copilot reads them locally regardless of host). Use your platform's branch policies / merge-request approval rules to require review on these paths. If your platform supports CODEOWNERS-style mandatory reviewers, point them at the customization files.

The principle is platform-independent: **anything that changes how the agent behaves should be reviewed like code.**

---

## 6. Treating agent-authored work like any other contribution

The single most consistent recommendation in the Well-Architected guidance: **agent-generated code goes through the same gates as human-generated code.** Same CI checks, same security scans, same code review, same merge requirements.

The temptation to exempt agent PRs ("it's just a small refactor", "the agent already followed our instructions") is the opening for the failure modes everyone fears. Agents make plausible-looking mistakes at scale. The review gates are the backstop.

A practical layered review setup:

| Layer | What it catches |
|---|---|
| Pre-tool hooks (Module 5) | Deterministic violations — never edit `migrations/*` directly, never write to a forbidden path |
| Code review agent + custom instructions | First-pass review for convention violations, missing tests, common security issues |
| CODEOWNERS + required human review | Domain judgment, architectural fit, anything subjective |
| CI: linting, typing, tests, security scans, dependency checks | Mechanical correctness, regressions, supply-chain issues |
| (Optional) Audit log streaming alerts | Anomalous patterns at the org level — agent modifying files outside its scope, ruleset-bypass attempts |

The point isn't that every PR needs every layer — it's that *you choose the layers per risk level* (high-risk repos get all of them; low-risk repos get fewer) and you don't quietly skip layers for agents.

---

## 7. Observability: knowing what agents did

Agents act faster than humans and at greater scale. If you can't see what they did, you can't course-correct.

Two surfaces, with different jobs:

**Audit log streaming** *(GitHub Enterprise)* — sends agent-related events (session creation, PR activity, policy changes) to your SIEM (Splunk, Sentinel, Datadog, etc.) for long-term retention and correlation. The key field is `agent_session_id` — it lets you trace one agent run end-to-end. Useful alerts:

- Agent sessions per user per day (anomaly = compromised account or runaway script)
- Agent changes to high-risk files (workflows, MCP config, instructions)
- Ruleset bypass attempts by an agent
- MCP policy changes

**Session monitoring** — the actual transcript of what the agent did, available in the GitHub UI. Use it for:

- Triaging a failed PR (which step went wrong?)
- Spot-checking high-risk repos periodically
- Validating new custom instructions or MCP configs after a change

Critical detail: **session transcripts are UI-only.** They cannot be streamed to a SIEM or pulled via API. So the audit log is for retention and detection; transcript review is for human investigation.

**Non-Enterprise teams:** you don't get audit log streaming for free. The fallback is your existing CI/audit pipeline plus webhook listeners for PR events. You can detect "agent submitted a PR that modified `.github/`" without GitHub Enterprise — you just have to wire it up. For session-level transcripts, what you see in your IDE is what you get; periodic team show-and-tell on interesting Copilot sessions is the human equivalent.

---

## 8. Cost: keeping spend predictable and attributable

Two things make agent cost different from regular Copilot cost:

- **Cloud agent sessions consume runner minutes** — every PR an agent opens runs CI, plus the agent's own work runs on a runner.
- **Premium model usage compounds** — different models have different request multipliers; an agent that uses an expensive model on every PR can burn budget fast.

The minimum cost-governance setup:

- **Spending limits** at the organization or cost-center level, with "stop usage at limit" enabled for hard caps. Without this, overruns are discovered after they happen.
- **Spend alerting** to the team responsible for each cost center, well below the hard cap.
- **Model selection guidance** — make it clear which models are appropriate for which tasks. A reviewer agent doing pattern matching probably doesn't need the most expensive model.
- **Adoption + consumption tracking** in the same dashboard. Cost without adoption context is just panic; cost trended against adoption tells you whether each marginal dollar is producing value.

**For non-Enterprise teams:** Copilot per-seat licensing is straightforward; the cost shape gets more complex when you start running CI-side automations that mimic agentic workflows (Module 6). Apply the same discipline: budget per pipeline, alerting on minute consumption, model selection guidance for whatever API you're calling.

---

## 9. Roles: who actually owns this

Even with the right structure, governance fails without ownership. Three roles tend to emerge:

- **Enterprise / platform team** — owns the floor: model allowlist, audit streaming, spending caps, the `.github-private` agents, enterprise-wide policy. Small team, slow change cadence.
- **AI managers** *(GitHub Enterprise concept; equivalent role exists informally elsewhere)* — delegated day-to-day stewardship of agents, instructions, and MCP configs. Larger group, faster cadence. They review pull requests against agentic primitives, evaluate new MCP servers, curate the shared instruction library.
- **Team leads / repository maintainers** — own the repo-level configuration. They write the team's `copilot-instructions.md`, decide which plugins to install, choose the prompts and skills that fit their codebase.

The split avoids two failure modes: enterprise owners drowning in trivial requests, and individual developers shipping unreviewed agent changes that affect everyone. Pushing decisions to the right level is the *whole* point of the tier model.

For non-Enterprise teams: the same three roles still make sense as job descriptions even if the platform doesn't enforce them. Designate a platform owner, designate per-business-unit AI stewards, and let team leads own repo-level config.

---

## 10. Common governance mistakes worth avoiding

- **Over-centralizing.** Pushing every preference into enterprise or org instructions creates token bloat and friction without adding value. Push down by default; push up only what genuinely must be uniform.
- **Under-governing.** Letting any developer add any MCP server without review is how an unsanctioned tool starts touching customer data. Rulesets on `mcp.json` (or equivalent platform controls) are the minimum.
- **Exempting agent PRs from review.** If a check applies to humans, it applies to agents. Especially security scans and required reviewers.
- **No visibility into what loaded.** Without audit log streaming (or its non-Enterprise equivalent), you find out about anomalies from outage reports, not from monitoring.
- **No spending limits.** "We'll watch usage" doesn't survive contact with a misconfigured workflow that opens 200 PRs in an afternoon.
- **Treating shared customizations as fire-and-forget.** A shared skill written 18 months ago for an API that has since changed will mislead the model. Shared customizations need an owner and a review cadence — quarterly at minimum.
- **Skipping the pilot.** A new enterprise-wide custom agent should land in a sandbox repo and a few volunteer teams before going to everyone. Roll out, measure, expand.
- **Ignoring the 4,000-character review limit** (Appendix). Every line you add to repo instructions for the sake of "governance" eats into the budget that Copilot code review will actually see.

---

## 11. A minimum viable governance setup

If your organization is starting from zero, the smallest thing that addresses the most risk is:

1. **At the enterprise/platform level:** enable audit log streaming, set a model allowlist, enable spending limits with hard caps, designate AI managers.
2. **At the org level:** publish a 30-line baseline `copilot-instructions.md` covering security non-negotiables and compliance language; protect agentic primitive files with rulesets/branch policies.
3. **For shared knowledge:** create one shared-standards repo with starter instructions and one or two reference skills; teams reference or copy from it.
4. **At the repo level:** team leads own their `.github/copilot-instructions.md`, scoped instructions, prompts, skills, and any MCP configs. Same review discipline as application code.
5. **In review process:** apply CI checks and security scans to agent PRs identically to human PRs; require human review on anything that touches `.github/` or MCP configs.

Everything else — enterprise custom agents, automatic code review, plugin marketplaces, SIEM dashboards, eval harnesses — is a layer you add when the underlying need shows up. Don't pre-build all of it. The minimum-viable setup catches most of the failure modes; the rest you grow into.

---

## 12. Where this fits in the course

Modules 1–6 taught the building blocks. Module 7 showed how to compose them on one realistic codebase. This module zoomed out to the org level: how to keep many such codebases consistent, safe, and affordable without becoming the bottleneck.

The thread through all of it: **match the level of control to the level the rule actually applies at.** Personal preference → personal instructions. Project rule → repo. Compliance baseline → org or shared library. Enterprise-wide policy floor → enterprise (or its non-platform equivalent). Skip a level and you either smother teams with rules they don't need or leave gaps that bite you later.

The appendix that follows is the operational complement: how to tell, day to day, whether all of this is actually working.
