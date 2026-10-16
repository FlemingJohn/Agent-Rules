# Agent Rules

Rules for coding agents (Claude Code, Codex, Cursor, Copilot, Gemini CLI and
others) so the code they write is modular, simple and easy to read.

---

## What is here

    core/               rules for every project
      code-shape.md       small files, one thing per file, no comments
      naming.md           plain names, no abbreviations
      simple-logic.md     no clever code, no needless helpers
      git.md              one file per commit, subject line only
    languages/
      python.md
      typescript.md
      javascript.md
    databases/
      sql.md              plain SQL for both databases
      postgresql.md       only what differs in PostgreSQL
      mysql.md            only what differs in MySQL
    frameworks/
      frontend/
        react.md
        nextjs.md
      backend/
        node.md
        express.md
        fastapi.md
      ai/
        langgraph.md

Every project uses all of `core/`, plus its language, its database and its
frameworks.

| Project | Files besides `core/` |
|---|---|
| Next.js app | `typescript.md`, `react.md`, `nextjs.md`, `sql.md`, and `postgresql.md` or `mysql.md` |
| React app with no Next.js | `typescript.md`, `react.md` |
| Express API | `typescript.md`, `node.md`, `express.md`, `sql.md`, and `postgresql.md` or `mysql.md` |
| FastAPI service | `python.md`, `fastapi.md`, `sql.md`, and `postgresql.md` or `mysql.md` |
| LangGraph agent | `python.md` or `typescript.md`, `langgraph.md` |
| Plain JavaScript project | `javascript.md`, plus its frameworks |

---

## How to use it in a project

**1. Copy the files you need** into the project, under `docs/agent-rules/`.
For a FastAPI service:

    docs/agent-rules/
      core/code-shape.md
      core/naming.md
      core/simple-logic.md
      core/git.md
      languages/python.md
      databases/sql.md
      databases/postgresql.md
      frameworks/backend/fastapi.md

**2. Make `AGENTS.md`** at the project root. Most agent tools read this file.

```markdown
# Rules

Read and follow every file below before writing any code.

@docs/agent-rules/core/code-shape.md
@docs/agent-rules/core/naming.md
@docs/agent-rules/core/simple-logic.md
@docs/agent-rules/core/git.md
@docs/agent-rules/languages/python.md
@docs/agent-rules/databases/sql.md
@docs/agent-rules/databases/postgresql.md
@docs/agent-rules/frameworks/backend/fastapi.md
```

**3. Make `CLAUDE.md`** at the project root, holding one line:

```markdown
@AGENTS.md
```

Claude Code reads `CLAUDE.md`, follows `@AGENTS.md`, and loads every file it
lists. Other tools read `AGENTS.md` and open the listed files. The rules are
written once and every agent sees the same ones.

---

## How each rules file is laid out

1. **The rules:** a short numbered list.
2. **Naming:** a table of what to call each kind of thing.
3. **Folder layout:** a real example tree.
4. **Examples:** real code, and good next to bad.
5. **Before you commit:** a yes or no checklist the agent checks its work
   against.

Language and framework files never repeat `core/`. They cover only what is
special to them.
