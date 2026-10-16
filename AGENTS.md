# Agent rules index

Do not read every rules file. Read the core files, then only the files that
match the work in front of you. Paths are relative to the folder this file is
in.

If two files disagree, the more specific one wins: a framework file beats a
language file, and a language file beats a core file.

---

## Always read

| File | What it covers |
|---|---|
| `core/code-shape.md` | small files, one thing per file, no comments |
| `core/naming.md` | plain names, no abbreviations |
| `core/simple-logic.md` | no clever code, no needless helpers |

---

## Read when the work matches

| When you are | Read |
|---|---|
| Writing or changing `.py` files | `languages/python.md` |
| Writing or changing `.ts` or `.tsx` files | `languages/typescript.md` |
| Writing or changing `.js`, `.jsx` or `.mjs` files | `languages/javascript.md` |
| Writing SQL, in a `.sql` file or inside code | `databases/sql.md` |
| Using PostgreSQL (`pg`, `psycopg`, `postgres://` in settings) | `databases/sql.md`, `databases/postgresql.md` |
| Using MySQL (`mysql2`, `pymysql`, `mysql://` in settings) | `databases/sql.md`, `databases/mysql.md` |
| Writing React components | `frameworks/frontend/react.md` |
| Working in a Next.js app (`next` in `package.json`) | `frameworks/frontend/react.md`, `frameworks/frontend/nextjs.md` |
| Writing Node server code | `frameworks/backend/node.md` |
| Working in an Express app (`express` in `package.json`) | `frameworks/backend/node.md`, `frameworks/backend/express.md` |
| Working in a FastAPI app (`fastapi` in `pyproject.toml`) | `frameworks/backend/fastapi.md` |
| Building a LangGraph graph (`langgraph` imports) | `frameworks/ai/langgraph.md` |
| Making a git commit | `core/git.md` |

A task can match more than one row. Read every row it matches, and nothing
else.
