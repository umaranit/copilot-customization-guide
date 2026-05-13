# Module 4: Skills — Packaged Domain Knowledge

> Part of [Designing Scalable, Reusable GitHub Copilot Customizations](README.md).
>
> **What this module is.** A guide to writing skills the model will actually reach for, and structuring them so they stay useful as they grow. For folder layout details, manifest fields, and packaging steps, see the [VS Code agent skills docs](https://code.visualstudio.com/docs/copilot/copilot-customization) and the examples in [`github/awesome-copilot/skills/`](https://github.com/github/awesome-copilot/tree/main/skills).

## 1. What makes a skill different

A skill looks a lot like a long instruction or a prompt — but the activation is what sets it apart.

- An **instruction** is loaded on every request, whether it's relevant or not.
- A **prompt** is loaded when *you* type its slash command.
- A **skill** is loaded when the *model* decides its trigger description matches the task at hand.

That last bit is the whole point. A skill's body — the actual knowledge — stays out of context until needed. Only its **description** is always loaded. The description is what the model reads to decide whether to pull in the body.

So a skill is really two things glued together:

1. A **trigger description** — short, eager, the load-bearing piece.
2. A **body** — bigger, lazy, the actual content.

If the description doesn't trigger, the body never runs. That's why most of the design effort goes into the description.

## 2. Writing trigger descriptions that actually fire

A description must answer two questions for the model: **when should I use this** and **when should I not?** Phrase those explicitly.

A description is a *trigger*, not a summary. Compare:

- ❌ *"Maps internal exceptions to HTTP responses for the payments-api."* — describes the body. The model has nothing to match a user task against.
- ✅ *"USE WHEN writing or reviewing controllers, error middleware, or OpenAPI error schemas in the payments-api. DO NOT USE FOR client-side error display or logging."* — clear conditions the model can match against the current task.

Two patterns work well:

- **Concrete tasks the user might be doing.** "Writing a Stripe webhook handler", "adding a database migration", "translating a React component."
- **File or symbol cues.** "Working in `src/api/controllers/`", "editing files that import `@stripe/sdk`."

Two patterns to avoid:

- **Pure topic descriptions.** "Stripe integration knowledge" tells the model *what* but not *when*.
- **Overlapping triggers.** Two skills both claiming "anything to do with the API" will compete unpredictably. Make each one's domain distinct.

## 3. What goes in the body — and what doesn't

The body is where the real content lives. Treat it like a focused playbook, not a README.

| Belongs in the body | Belongs somewhere else |
|---|---|
| Step-by-step procedures | One-line repo conventions → instructions |
| Mapping tables, decision trees | Personal style preferences → personal instructions |
| Templates the model should follow | User-triggered workflows → prompts |
| Pointers to related files in the skill folder | General docs → repo `README.md` |

A skill body should answer "*how do I do this thing?*" in enough detail that the model can act on it. If it's just background reading, it doesn't help — the model needs concrete patterns to copy.

*Example.* A `db-migration` skill body has: the file naming convention, a template for `up`/`down` SQL, the command to test rollback locally, and a pointer to `./templates/migration.sql`. It does *not* have a history of why the team chose this pattern — that's a doc, not a skill.

## 4. When to use a folder, not just a file

A skill can be a single `SKILL.md`, or a folder with supporting files. Use a folder when:

- You have **templates** the model should copy (e.g., a starter file, a config snippet).
- You have **scripts** the model can run (e.g., a generator, a validator).
- You have **reference material** worth keeping next to the skill (e.g., a mapping table, an example).

A typical multi-file skill layout:

```
skills/db-migration/
├── SKILL.md            # description + body
├── templates/
│   ├── migration.sql
│   └── seed.sql
└── examples/
    └── add-column.md
```

The body of `SKILL.md` references those files by path. The model only opens them when the workflow calls for it, so they don't bloat context.

If your skill has no templates, scripts, or references, keep it a single file. Folders are a tool, not a default.

## 5. Composing skills with prompts and instructions

Skills are most powerful when they slot in alongside the other primitives:

- **Repo instructions** carry the universal rules (language, layout). Skills assume those are already known.
- **Prompts** can rely on skills triggering automatically. A `/add-endpoint` prompt doesn't need to explain HTTP error mapping if the `api-error-mapping` skill is going to fire.
- **Other skills** can be referenced from a skill body when one workflow leads into another.

The mental model: instructions set the *baseline*, skills add *expertise on demand*, prompts let users *kick off* multi-step work. They overlap deliberately, not redundantly.

*Example.* A user types "*Add a `DELETE /users/:id` endpoint.*" The repo instructions tell the model the framework and folder. The `api-error-mapping` skill triggers because the model is editing a controller. The `db-migration` skill does *not* trigger — no schema change is involved. No prompt was invoked. Three primitives quietly cooperate; only the relevant ones spend context.

## 6. Common mistakes worth avoiding

- **Descriptions that read like a table of contents.** If your description summarizes the body, it's not a trigger. Rewrite it as "USE WHEN…" / "DO NOT USE FOR…".
- **Skills that should have been instructions.** A 3-line "always do X" rule doesn't need a skill — it's eager content with no body worth hiding.
- **Skills that should have been prompts.** If you keep typing a slash command to make the skill fire, the model never wanted to trigger it. Make it a prompt.
- **Overlapping triggers.** Two skills competing for the same task → the model picks one unpredictably. Make each skill's domain crisp.
- **Folder-without-need.** A skill with one file in it doesn't need a folder. Don't pre-build structure you're not using.
- **Bodies that explain instead of instruct.** Background and history don't help the model act. Patterns, templates, and steps do.

## 7. What to carry into the next module

- Skills are the **on-demand knowledge layer**. The description is eager; the body is lazy. Most of the design work goes into the description.
- A skill earns its keep when its body would be too long for an instruction *and* its trigger doesn't depend on the user typing a command.
- The next module covers **custom agents**, **MCP**, and **hooks** — three primitives that change *how* and *what* the model can do, rather than *what it knows*.
