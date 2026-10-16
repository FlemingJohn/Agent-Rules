# Writing Express

Builds on `core/`, `languages/typescript.md` and
`frameworks/backend/node.md`. This file covers only what is special to
Express.

Written for Express 5 with plain SQL. The examples use PostgreSQL through
`pg`. For MySQL, see `databases/mysql.md`.

The examples use one made-up API throughout: tasks that can be created and
read.

---

## The rules

1. **Thin handlers.** A handler reads the input, calls one or two functions
   and sends the reply. About fifteen lines at most.
2. **One router per area.** `taskRouter.ts` holds every `/tasks` route. The
   router is the one thing in that file.
3. **Check every body with zod.** `NewTask.parse(req.body)`. Never trust
   `req.body` as it is.
4. **No `try` and `catch` in handlers.** Express 5 sends any thrown error, or
   any rejected `await`, to the error handler for you.
5. **One error handler.** It turns known errors into the right status code,
   and everything else into a 500.
6. **Query functions know nothing about HTTP.** They take plain values and
   return checked data. No `req` or `res` inside them.
7. **SQL is plain and has parameters.** Never build a query with a template
   string.
8. **`req`, `res` and `next` keep their short names.** Express uses them
   everywhere, so they count as names the outside world already uses.

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Router | `taskRouter` | area then `Router` |
| Query function | `selectTaskById`, `insertTask` | starts with the SQL verb |
| Query text | `SELECT_TASK_BY_ID` | the function name in `UPPER_SNAKE` |
| Middleware | `handleError`, `checkSignedIn` | verb then noun |
| App builder | `makeApp` | starts with `make` |
| Request body schema | `NewTask` | what the caller sends |
| Reply schema | `Task` | the noun |

Paths are plural nouns in lower case: `/tasks`, `/tasks/:taskId`.

---

## Folder layout

    project/
      package.json
      .env
      src/
        main.ts
        makeApp.ts
        settings.ts
        handleError.ts
        database/
          pool.ts
        tasks/
          Task.ts
          NewTask.ts
          TaskStatus.ts
          selectTaskById.ts
          insertTask.ts
          taskRouter.ts
      migrations/
        001_create_tasks.sql

---

## Building and starting the app

`src/makeApp.ts`

```ts
import express from "express";
import { handleError } from "./handleError.ts";
import { taskRouter } from "./tasks/taskRouter.ts";

export function makeApp() {
  const app = express();
  app.use(express.json());
  app.use("/tasks", taskRouter);
  app.use(handleError);
  return app;
}
```

`src/main.ts`

```ts
import { makeApp } from "./makeApp.ts";
import { settings } from "./settings.ts";

makeApp().listen(settings.port, () => {
  console.log(`Listening on port ${settings.port}`);
});
```

The error handler always goes last.

---

## The database pool

`src/database/pool.ts`

```ts
import pg from "pg";
import { settings } from "../settings.ts";

export const pool = new pg.Pool({ connectionString: settings.databaseUrl });
```

---

## Query functions

`src/tasks/selectTaskById.ts`

```ts
import { pool } from "../database/pool.ts";
import { Task } from "./Task.ts";

const SELECT_TASK_BY_ID = `
  SELECT id, title, status, created_at AS "createdAt"
  FROM tasks
  WHERE id = $1
`;

export async function selectTaskById(taskId: number) {
  const result = await pool.query(SELECT_TASK_BY_ID, [taskId]);
  const row = result.rows[0];
  if (!row) {
    return null;
  }
  return Task.parse(row);
}
```

`src/tasks/insertTask.ts`

```ts
import { pool } from "../database/pool.ts";
import type { NewTask } from "./NewTask.ts";
import { Task } from "./Task.ts";

const INSERT_TASK = `
  INSERT INTO tasks (title)
  VALUES ($1)
  RETURNING id, title, status, created_at AS "createdAt"
`;

export async function insertTask(newTask: NewTask) {
  const result = await pool.query(INSERT_TASK, [newTask.title]);
  return Task.parse(result.rows[0]);
}
```

Every row goes through the zod schema, so the rest of the code gets data it
can trust.

---

## The router

`src/tasks/taskRouter.ts`

```ts
import { Router } from "express";
import { insertTask } from "./insertTask.ts";
import { NewTask } from "./NewTask.ts";
import { selectTaskById } from "./selectTaskById.ts";

export const taskRouter = Router();

taskRouter.get("/:taskId", async (req, res) => {
  const task = await selectTaskById(Number(req.params.taskId));
  if (!task) {
    res.status(404).json({ message: "Task not found" });
    return;
  }
  res.json(task);
});

taskRouter.post("/", async (req, res) => {
  const newTask = NewTask.parse(req.body);
  const task = await insertTask(newTask);
  res.status(201).json(task);
});
```

If `parse` fails, it throws. Express 5 passes that to the error handler.

---

## The error handler

`src/handleError.ts`

```ts
import type { ErrorRequestHandler } from "express";
import { ZodError } from "zod";

export const handleError: ErrorRequestHandler = (error, _req, res, _next) => {
  if (error instanceof ZodError) {
    res.status(400).json({ message: "Invalid input", issues: error.issues });
    return;
  }
  console.error(error);
  res.status(500).json({ message: "Something went wrong" });
};
```

It must take four arguments, or Express does not see it as an error
handler.

---

## Express 5 changes worth knowing

- Thrown errors and rejected promises in handlers reach the error handler on
  their own.
- A wildcard path needs a name: `/files/*filePath`, not `/files/*`.
- `req.query` is read-only.
- `res.status(...)` only takes a whole number from 100 to 999.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    NewTask.parse(req.body)               insertTask(req.body)
    throw, and let handleError reply      try { ... } catch (e) { res... }
    WHERE id = $1                         `WHERE id = ${taskId}`
    one taskRouter.ts per area            every route in makeApp.ts
    selectTaskById(taskId)                selectTaskById(req)
    res.status(404).json(...); return;    res.status(404).json(...)
                                          res.json(task)
    app.use(handleError) last             app.use(handleError) first

---

## Before you commit

- Is every handler about fifteen lines or less?
- Does every body go through a zod schema before it is used?
- Is there a `try` and `catch` in a handler?
- Is every query parameterised, with no template-string values?
- Does any query function take `req` or `res`?
- Does every early reply end with `return`?
