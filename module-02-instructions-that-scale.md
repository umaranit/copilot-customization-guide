# Module 2: Instructions That Scale

> Part of [Scalable & Reusable Agent Design in GitHub Copilot](README.md).
>
> **What this module is.** A practical guide to writing Copilot instructions that stay useful as your repo, team, and codebase grow. For file paths, frontmatter fields, and precedence rules, see the [GitHub Copilot docs](https://docs.github.com/en/copilot/customizing-copilot/about-customizing-github-copilot-chat-responses) and the [VS Code docs](https://code.visualstudio.com/docs/copilot/copilot-customization).

## 1. Three places instructions can live

Before deciding *what* to write, decide *where* it belongs. Instructions can sit at three levels:

| Level | Who sets it | Who sees it | Use it for |
|---|---|---|---|
| **Personal** | You | Only you | Your individual style ("respond in Portuguese", "explain one concept per line") |
| **Repository** | Repo maintainers | Everyone working in this repo | Project-wide rules — language, frameworks, conventions, file layout |
| **Organization** | Org owners | Everyone in the org | Company-wide policy — preferred language, security guidelines, escalation paths |

When several apply at once, **personal wins, then repository, then organization**. They don't replace each other — they all reach the model — but if they conflict, that's the order Copilot favors.

*Example.* You set a personal instruction "always reply in Portuguese." Your repo says "respond in English." You'll get Portuguese, because personal beats repository.

The takeaway: don't push personal preferences into repo instructions, and don't push project rules into your personal settings. Each level has a clear audience.

> **If you're not on GitHub Enterprise/Business.** The organization tier is a GitHub platform feature — only org owners on Enterprise or Business plans can set instructions that reach every member automatically. Teams on Azure DevOps, GitLab, or Bitbucket usually approximate this with a shared "starter" repo, a template repo, or a plugin (Module 6) distributed through their internal channels. The pattern is: write the org-wide rules once, then have every consuming repo include them — either by copying the file or by depending on the plugin. Less automatic than the platform tier, but the governance shape is the same.

## 2. Repo-wide vs. scoped: when to split

Inside a repo, you have two choices:

- **Repo-wide** — `copilot-instructions.md`, loaded on every request.
- **Scoped** — `*.instructions.md` files with an `applyTo` glob, loaded only when an attached file matches.

The decision is just about **how often the rule actually applies**:

- *Applies to every file in the repo?* → Repo-wide.
- *Only matters for one language, framework, or folder?* → Scoped.

A rule earning its place repo-wide should be short and almost always relevant. Anything narrower belongs in a scoped file.

- *Repo-wide example* — "Use TypeScript 5.4. Tests live next to source as `*.test.ts`. Never import from `src/internal/**`." Three lines, true for every file.
- *Scoped example* — A `react.instructions.md` with `applyTo: src/web/**/*.tsx` saying "Functional components only. Use `clsx` for class names. Prefer `useSyncExternalStore` over custom subscriptions." Useful in `src/web`, irrelevant in `src/server`.

**Rule of thumb:** if you find yourself writing "when working on X…" inside `copilot-instructions.md`, that paragraph wants to be a scoped file with `applyTo: X`.

## 3. Writing globs that don't quietly bloat context

The whole point of a scoped instruction is that it stays out of context when it's not needed. Globs are how you control that.

Two common slip-ups:

- **`applyTo: "**/*"`** — matches everything, which is the same as repo-wide but harder to find. If a rule really applies everywhere, put it in `copilot-instructions.md` and own it.
- **Overlapping globs** — three files all matching `src/web/**` will all load on the same request. That's fine if they're each short and distinct, but watch the total.

Aim for the **narrowest glob that captures the rule's domain**:

- Frontend React → `src/web/**/*.{ts,tsx}`
- Database migrations → `db/migrations/**/*.sql`
- Public API surface → `src/api/public/**`

For monorepos, if you open a sub-folder in VS Code instead of the repo root, scoped instructions at the root won't load by default. Enable `chat.useCustomizationsInParentRepositories` so VS Code walks up to find them.

## 4. `AGENTS.md` when portability matters

`AGENTS.md` is a more portable format for repo instructions. The same file can be picked up by Copilot, Claude, Gemini, and other agents that read it.

Use it when:

- Your repo is consumed by people on more than one AI tool.
- You want a single source of truth for "how to work in this repo."

Stick with `copilot-instructions.md` when:

- You're Copilot-only and want the conventional location.
- You need Copilot-specific behavior that other tools won't understand.

Either way, the content rules are the same — short, broadly applicable, authoritative.

## 5. Common mistakes worth avoiding

- **The everything-instruction.** A 400-line `copilot-instructions.md` that pays its full token cost on every request, even ones it has nothing to say about. Move the long bits into skills (lazy) and the file-specific bits into scoped files (gated).
- **Conflicting rules across levels.** Personal says "use semicolons", repo says "no semicolons." Personal wins, but the inconsistency confuses everyone. Decide once where each rule belongs and remove duplicates.
- **Personal preferences in repo files.** "Always greet me by name" doesn't belong in `copilot-instructions.md` — it's personal, not project policy.
- **Forgetting the 4,000-character limit on Copilot code review.** PR reviews only read the first ~4,000 chars of an instruction file. If your most important rules are at the bottom, they won't be applied at review time.
- **Writing instructions like docstrings.** "This file describes our coding standards" is metadata. Just state the rule directly: "Use camelCase for variables."

## 6. What to carry into the next module

- Instructions are the cheapest primitive to *write* and the most expensive to *load*. Every line you add ships on every request — make sure each one earns its keep.
- Two questions answer most placement decisions: **who is the audience** (you, the repo, the org) and **how often does the rule apply** (always, or only for some files).
- When a rule is long, task-specific, or only useful when the user explicitly asks for it, it isn't really an instruction — it's a prompt or a skill. Module 3 covers prompts; Module 4 covers skills.
