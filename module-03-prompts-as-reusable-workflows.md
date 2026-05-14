# Module 3: Prompts as Reusable Workflows

> Part of [Designing Scalable, Reusable GitHub Copilot Customizations](README.md).
>
> **Prerequisites.** [Module 1](module-01-customization-primitives.md) (loading-rule cheat sheet); [Module 2](module-02-instructions-that-scale.md) (so you don't restate repo rules inside prompts).
>
> **What this module is.** A guide to turning recurring chat asks into reusable, parameterized workflows — and keeping them thin enough to stay maintainable. For frontmatter syntax, slash-command setup, and `/create-prompt` scaffolding, see the [VS Code prompt files docs](https://code.visualstudio.com/docs/copilot/copilot-customization).
>
> **Cost.** Loaded **on user invoke** (when you type `/name`). Zero cost when not invoked. The discipline here is keeping prompts thin so the per-invoke cost stays small.

## 1. When does something deserve to be a prompt?

A good rule: **the third time you type a similar message into chat, make it a prompt.**

Before that, the cost of writing and maintaining a prompt outweighs the savings. After that, you're paying the cost of re-typing it (and getting subtle variations every time) on every use.

Signs something is ready to become a prompt:

- You've copy-pasted nearly the same chat message more than twice.
- The ask has a clear *shape* — a couple of slots that change (a file, a name) and a body that doesn't.
- You'd want a teammate to be able to run the same workflow without learning your phrasing.

If only one of those is true, leave it as freeform chat. If all three are, write a prompt.

## 2. Anatomy of a thin prompt

A prompt should hold the **workflow** — the steps, the shape of the output, the tool choices. It should *not* hold things that belong elsewhere:

| Belongs in the prompt | Belongs somewhere else |
|---|---|
| The steps to perform | Repo conventions → instructions |
| The shape of the output | Domain knowledge → skills |
| Which tools may be used | Personal style preferences → personal instructions |
| The arguments the user provides | Long examples → linked files |

A good prompt reads like a short recipe with placeholders, not a wall of text. If yours is more than a screen long, something else is masquerading as a prompt.

*Example.* A `/add-test` prompt for generating tests should say *"write a test for `${file}`, mirror the path, cover happy + edge + error case, use AAA blocks."* It should **not** restate "we use Vitest" — that already lives in the repo instructions.

## 3. Choosing the agent

Prompts run under one of three built-in agents (set via the `agent` field in frontmatter, or inherited from the active chat agent). The choice shapes what the prompt can actually do:

- **`ask`** — chat-only. The model talks back; nothing in your workspace changes. Use for analysis, explanations, Q&A about the codebase.
- **`plan`** — research and produce a structured implementation plan, with clarifying questions, but stop short of making changes. Use when you want a reviewed plan before any code is written.
- **`agent`** — full tool access (read, write, run, search). Use for multi-step workflows that touch several files, need to inspect the codebase, or carry the work all the way to a working change.

A simple decision rule:

- *Read-only Q&A?* → `ask`
- *Want a plan before any change?* → `plan`
- *Anything that actually edits files or runs tools?* → `agent`

Defaulting everything to `agent` works but gives the model more rope than it usually needs. Pick the smallest agent that gets the job done.

> **Note.** The older `edit` mode is deprecated and hidden by default in current VS Code — Agent mode now covers focused single-file edits as well as multi-file work. If you still see references to `edit` in older prompts or examples, treat them as `agent`. The setting `chat.editMode.hidden` can restore it temporarily if a team needs it during migration. Older docs also called these "chat modes"; they're now called **agents**, and custom variants live in `.agent.md` files (see Module 5).

## 4. Restricting tools — guardrails on purpose

In `agent` mode, you can list which tools the prompt is allowed to call. This isn't about security theater — it's about **scoping the model's behavior to the workflow**.

When to restrict tools:

- The workflow only needs to read and write specific kinds of files. Limit it to those tools.
- You want the prompt to *plan* but not *run* (e.g., a refactor proposer). Drop the file-write tools.
- You want repeatable behavior. Fewer tools means fewer paths the model can take.

When to leave tools open:

- Exploratory or one-off prompts where you don't yet know what's needed.
- Workflows that legitimately need broad access (a "fix this failing test" prompt may need to read, run, edit, and search).

*Example.* `/add-test` only needs to read source, search the repo for conventions, and create the test file → restrict to `read_file`, `grep_search`, `create_file`. The model can't accidentally run a build or modify unrelated files.

## 5. Designing arguments

Keep arguments **few and obvious**. Most prompts need zero or one. A handful need two. If you're reaching for a third, you're probably building a config form, not a prompt.

- Use `${file}` for the attached file — the most common single argument.
- Use `${input:name}` for free-text arguments the user fills in at runtime.
- Default to no arguments when the prompt can infer everything from context.

A prompt with five arguments is a sign the workflow is too generic. Either split it into two more specific prompts or move the variation into the prompt body as a question the model asks the user.

## 6. Composing with instructions and skills

Prompts are most powerful when they **delegate**, not when they duplicate.

- Project conventions (test runner, file layout, naming) → already in repo instructions; the prompt inherits them automatically.
- Domain expertise (how a Stripe webhook works, how to write a database migration) → put in a skill; the model will pull the body in if the prompt's task triggers it.
- Tool sequences (search, then read, then edit) → that's the prompt itself.

If you find your prompt explaining "how things work in this codebase," that explanation belongs in instructions or a skill, and the prompt should just point at the work.

*Example.* A `/migrate-column` prompt says *"add a database migration for the change described in `${input:change}`"*. The repo instructions already say what language and folder to use. A `db-migration` skill explains the up/down/rollback conventions. The prompt itself is three lines and stays that way.

## 7. Common mistakes worth avoiding

- **Prompts that restate the repo's rules.** Anything already in `copilot-instructions.md` doesn't need to be repeated. The model already has it.
- **Prompts that should have been skills.** If you find yourself typing `/skillName` every time the model is editing a controller, you wanted a skill — let the model trigger it.
- **Prompts in `agent` mode that only need `ask`.** Giving the model tools it doesn't need increases variance and slows things down.
- **Argument creep.** Five arguments turns a prompt into a fragile form. Two is the comfort zone; three is the ceiling.
- **One-shot prompts.** A workflow you'll run exactly once is just a chat message. Don't promote it.

## 8. Cheat sheet, with prompts in focus

The row from the [Module 1 cheat sheet](module-01-customization-primitives.md#5-the-decision-cheat-sheet) this module zooms into:

| If you want… | Reach for | Loading rule | Cost |
|---|---|---|---|
| A workflow *you* will run repeatedly via slash command | Prompt | On user invoke | **Zero unless invoked** |

Notice: prompts and custom agents share the *on user invoke* trigger. The difference is **persistence** — a prompt resets after one exchange; a custom agent keeps its persona across follow-ups. Module 5 covers that distinction.

## 9. What to carry into the next module

- Prompts are **user-invoked workflows**. They're for things you'll run again, where the steps are stable but the inputs vary.
- A good prompt is **thin**: it holds the workflow and delegates everything else (conventions to instructions, knowledge to skills).
- The next module covers **skills** — the same idea of packaged knowledge, but triggered by the *model* when a matching task appears, not by you typing a slash command.
