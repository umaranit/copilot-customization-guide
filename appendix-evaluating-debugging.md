# Appendix: Evaluating & Debugging Customizations

> Part of [Scalable & Reusable Agent Design in GitHub Copilot](README.md).
>
> **What this appendix is.** A practical reference for the part the rest of the course assumes: how to tell whether your customizations are actually working. Covers the built-in debug surfaces, the most common reasons primitives silently fail to fire, and a lightweight evaluation loop you can run in an afternoon. For exact UI paths and command names, see the [VS Code Customize AI docs](https://code.visualstudio.com/docs/copilot/copilot-customization) — they change faster than this document will.

A customization you can't observe is a customization you can't trust. The whole point of the loading-rule discipline from Module 1 is wasted if you can't tell whether a skill actually fired, an instruction was actually loaded, or a prompt actually used the tools you restricted it to. This appendix is about closing that loop.

---

## 1. Two surfaces that show you what's happening

VS Code ships two debug surfaces for Copilot customizations. Most of the diagnosis in this appendix uses one or both.

**Show Agent Debug Logs.** A panel that records, for each request, what was loaded into context: which instruction files, which skills' descriptions, which skills' bodies (if any fired), which tools were available, which were called. This is the source of truth for "did the thing I built actually run?" If the log doesn't mention your skill, the skill didn't fire — no amount of staring at the file will change that.

**Agent Customizations editor.** A view of every customization that's currently active for your workspace, with their source (personal / repo / org / plugin) and their loading rule. Useful for the inverse question: "what's running that I didn't intend?" Plugins from a marketplace, an `AGENTS.md` from a parent repo, leftover personal instructions from years ago — they all show up here.

The pattern is: when something is wrong, check the debug log first to see what actually loaded; check the Customizations editor second to see what *could* have loaded.

---

## 2. The most common failure: a skill that never fires

By a wide margin, the most common debug call is "I wrote a skill, the model isn't using it." Almost always the cause is the description, not the body.

**The model decides whether to read a skill body based only on its description.** If the description doesn't match how the user phrases their question, the body never enters context, and the skill might as well not exist.

**Symptoms in the debug log:**

- Skill listed under "available skills" — good, it was advertised.
- Skill body *not* listed under "loaded into context" — the model didn't choose it.

**Common causes:**

- Description is too vague. "Helps with database stuff" doesn't fire when the user asks "how do I add a new column to the users table." "USE WHEN the user is writing or modifying database migration files, especially Postgres ALTER TABLE statements" does.
- Description doesn't mention the user's vocabulary. If your team says "schema change" but the description says "migration", a question about a "schema change" may not match.
- Description tries to do too much. A skill that claims to handle migrations *and* query performance *and* connection pooling will fire unreliably for all three. Split it.
- Multiple skills overlap. Two skills with similar descriptions can confuse the model into picking the wrong one — or neither.

**The fix is iterative.** Try the question, check the log, adjust the description, try again. Three to five rounds is normal. If you find the right description, save the working examples — they're the eval set for the next section.

---

## 3. The next most common failure: scoped instructions that don't apply

Scoped instructions (`applyTo: "**/*.ts"`) only load when the agent is touching a file that matches the glob. The two failure modes:

**Glob doesn't match what you think it matches.**

- `*.test.ts` matches `foo.test.ts` but not `tests/foo.ts`.
- `**/migrations/*.sql` matches `db/migrations/001.sql` but not `migrations/001.sql` (depending on glob style).
- `src/**/*.ts` excludes files outside `src/` — easy to forget when refactoring.

The debug log shows which scoped instruction files actually loaded for a given request. If yours isn't there, the glob is the suspect.

**The agent isn't touching any matching file yet.**

A scoped instruction for `**/*.test.ts` won't help when the user is asking "how should I structure tests?" if no test file is open. Scoped instructions help when files are read or written — they don't fire on abstract questions about those files. If you need the rule available for general questions, it belongs in repo-wide instructions, not a scope.

---

## 4. Conflicting instructions

When two instructions disagree, Copilot follows precedence: personal beats repository beats organization (Module 2). The trap is that both still load and both still consume tokens, and the precedence resolution isn't always visible to the user.

**Symptoms:**

- "I told it to use Portuguese and it's giving me English." → A scoped instruction or a custom agent overrode the personal preference. Check what loaded.
- "Half the time it follows the rule, half the time it doesn't." → Two instructions are partially in tension and the model is splitting the difference.

**The fix:**

- Read the debug log to see what loaded.
- Decide which level the rule actually belongs at. A team coding standard belongs at the repo level; a personal style preference belongs at the personal level. If a personal preference contradicts a team standard, that's a conversation, not a tooling problem.
- Delete redundant rules. Instructions that say similar things in different words make the model less consistent, not more.

---

## 5. The 4,000-character limit on Copilot code review

When Copilot does code review on a pull request, it reads the repo's instructions — but only up to **4,000 characters**. Anything past that is silently ignored.

This catches teams that grow their repo instructions over time. The first 4,000 characters might cover what matters; the rest — including, often, the most recently added rules — never reaches the reviewer.

**Symptoms:**

- "We added a rule three months ago and the code review still misses it."
- "The reviewer follows the old rules but ignores anything past a certain section."

**Fixes:**

- Keep repo-wide instructions tight. The discipline from Modules 1 and 2 — every line costs tokens, push detail into skills — also keeps you under the limit.
- Put the highest-priority rules first in the file.
- For long-form domain knowledge that the reviewer should reference, package it as a skill rather than expanding the instruction file. (Skills are loaded by code review when their descriptions match.)

---

## 6. MCP tools that look available but never get called

When you add an MCP server, its tools enter the eager tool list. The model knows they exist but won't call them unless the work needs them.

**If a tool is never called when you expect it to be:**

- Tool description is vague. The model picks tools the same way it picks skills — based on the description. "Run a SQL query" is weaker than "Run a read-only SQL query against the staging Postgres database; use this when you need to check what data actually looks like."
- Tool overlaps with the model's built-in capabilities. A "read file" MCP tool will lose to the built-in file reader; the model picks the simpler option.
- The model genuinely doesn't think the tool is needed. Sometimes that's correct — and the fix is to ask more specifically, not to add more tools.

**If a tool is called too often:**

- Description is too eager ("use this for any database question").
- Tool isn't scoped enough — a tool that can read *and* write becomes the default for both.

The debug log shows tool calls per request. A tool you've added that never appears in any log is a tool to remove.

---

## 7. A lightweight evaluation loop

For anything beyond a single repo, you'll eventually want to know: when I change a skill, did I make it better or worse? "It still works for me" doesn't scale.

A lightweight loop — runnable in an afternoon — looks like this:

**1. Build a small set of golden prompts.**

Ten to twenty representative questions, with what a correct answer looks like. For `payments-api` from Module 7:

- "Add a test for `chargeCustomer`." → Correct: uses the project's mock helper, not a raw `db` stub.
- "Change the `email` column to `varchar(320)`." → Correct: produces an up/down migration with the index.
- "Where do we handle webhook signature verification?" → Correct: names the right file.

These are not unit tests. They're judgment calls — but judgment calls you've written down, so they stay consistent across runs.

**2. Run them, save the outputs.**

Just run each prompt against the current setup. Save the responses. Note for each one whether it met the bar.

**3. Change one thing.**

Edit one skill description, tighten one instruction, add one prompt. Then re-run the same set.

**4. Compare.**

Same prompts, two runs, two sets of judgments. Did the change help, hurt, or do nothing? Watch for *regressions* especially — improvements on the thing you targeted that quietly broke something else.

This isn't rigorous evaluation. It's the difference between "I think it's better" and "I checked." That's enough for almost all team-scale work.

**A note on automation:** there are mature evaluation tools for AI systems (Promptfoo, the OpenAI evals harness, custom harnesses built on the model APIs). If your customization set is large or business-critical, invest in one. For most teams starting out, a Markdown file with twenty golden prompts and a half hour of comparison after each change is the right starting point. Add infrastructure when the manual loop becomes the bottleneck.

---

## 8. A debugging checklist

When something isn't working, in this order:

1. **Open the debug log.** What actually loaded for the failing request?
2. **Open the Customizations editor.** What customizations could have loaded?
3. **If the expected primitive didn't load:**
   - Skill body missing → description issue (Section 2)
   - Scoped instruction missing → glob issue (Section 3)
   - Tool never called → MCP description issue (Section 6)
4. **If the expected primitive *did* load but the answer is still wrong:**
   - The customization is firing but its content is wrong, conflicting, or insufficient. Read the file with fresh eyes.
   - Look for conflicts (Section 4) — another customization may be overriding it.
   - Check the 4,000-char limit if this is a code-review failure (Section 5).
5. **If you've fixed it once but want to keep it fixed:**
   - Add the failing prompt to your golden set (Section 7) so the next change can't silently re-break it.

Most debugging is one of these five steps. The exotic failure modes are rare; the boring ones (vague descriptions, wrong globs, instructions that drifted out of date) account for the vast majority of "why isn't this working?"

---

## 9. Closing thought

The course's one-sentence summary was: *match the loading rule to the cost, and pick the smallest primitive that solves the problem.* The appendix's one-sentence summary is its mirror: *if you can't see what loaded, you can't tell whether you matched it correctly.*

Build the customization. Check what fired. Adjust the description, the glob, the scope. Save the prompts that exposed the problem. Re-run them next time you change something. That's the whole loop, and it's enough to keep a real-world Copilot setup honest as it grows.
