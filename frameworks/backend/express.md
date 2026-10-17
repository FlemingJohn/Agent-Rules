# Writing Express

Builds on `core/`, `languages/typescript.md` and
`frameworks/backend/node.md`. This file covers only what is special to
Express.

Written for Express 5 with plain SQL. The examples use PostgreSQL through
`pg`. For MySQL, see `databases/mysql.md`.

The examples use one made-up API throughout: tasks that can be created and
read.

---

## The layers

A request passes through the same layers every time, in this order:

    request → middleware → route → service → query → database

| Layer | Its one job | Never |
|---|---|---|
| `middleware/` | checks every request before a route: signed in, logging, errors | business rules |
| `routes/` | read the request, check the body, call one service, send the reply | SQL, business rules |
| `services/` | the business rules: who may do what, what must be true | `req`, `res`, status codes |
| `queries/` | run one SQL statement and return checked data | decisions |
| `schemas/` | the shape of data coming in and going out | logic |
| `errors/` | named errors that services throw | logic |

Each layer calls only the layer below it. A route never calls a query. A
query never calls a service.

---

## The rules

1. **Thin routes.** A route reads the input, calls one service and sends the
   reply. About fifteen lines at most. No controller files: the handler sits
   in the route file.
2. **Services hold the rules.** They take plain values, return data, and
   throw a named error when something is wrong.
3. **Queries hold one SQL statement each.** Plain SQL with parameters. Never
   a template string with values in it.
4. **Check every body with zod in the route.** `NewTask.parse(req.body)`.
   Never pass `req.body` on as it is.
5. **No `try` and `catch` in routes or services.** Express 5 sends any thrown
   error to the error handler for you.
6. **One error handler.** It looks up the status code for each named error
   and sends the reply.
7. **Inside each layer, one folder per area.** `services/tasks/createTask.ts`,
   not one large `taskService.ts`.
8. **`req`, `res` and `next` keep their short names.** Express uses them
   everywhere.

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Route file | `taskRoutes.ts` exports `taskRoutes` | area then `Routes` |
| Middleware | `checkSignedIn`, `handleError` | verb then noun |
| Service | `getTask`, `createTask`, `closeTask` | verb then noun |
| Query | `selectTaskById`, `insertTask` | starts with the SQL verb |
| Query text | `SELECT_TASK_BY_ID` | the function name in `UPPER_SNAKE` |
| Schema | `Task`, `NewTask` | what the data is |
| Error | `NotFoundError`, `NotAllowedError` | what went wrong, then `Error` |
| App builder | `makeApp` | starts with `make` |

Services and queries never share a verb, so a service can call a query
without a name clash.

Paths are plural nouns in lower case: `/tasks`, `/tasks/:taskId`.

---

## Folder layout

    project/
      package.json
      .env
      src/
        main.ts
        makeApp.ts
        config/
          settings.ts
        database/
          pool.ts
        middleware/
          checkSignedIn.ts
          handleError.ts
        routes/
          taskRoutes.ts
        services/
          tasks/
            getTask.ts
            createTask.ts
        queries/
          tasks/
            selectTaskById.ts
            insertTask.ts
        schemas/
          tasks/
            Task.ts
            NewTask.ts
            TaskStatus.ts
        errors/
          NotFoundError.ts
          NotAllowedError.ts
      migrations/
        001_create_tasks.sql

Imports use the `#src/` path set up in `node.md`:
`import { pool } from "#src/database/pool.ts"`.

---

## Building and starting the app

`src/makeApp.ts`

```ts
import express from "express";
import { checkSignedIn } from "#src/middleware/checkSignedIn.ts";
import { handleError } from "#src/middleware/handleError.ts";
import { taskRoutes } from "#src/routes/taskRoutes.ts";

export function makeApp() {
  const app = express();
  app.use(express.json());
  app.use("/tasks", checkSignedIn, taskRoutes);
  app.use(handleError);
  return app;
}
```

`src/main.ts`

```ts
import { settings } from "#src/config/settings.ts";
import { makeApp } from "#src/makeApp.ts";

makeApp().listen(settings.port, () => {
  console.log(`Listening on port ${settings.port}`);
});
```

Building and starting are two files, so tests can build the app without
starting a server. The error handler always goes last.

---

## Route

`src/routes/taskRoutes.ts`

```ts
import { Router } from "express";
import { NewTask } from "#src/schemas/tasks/NewTask.ts";
import { createTask } from "#src/services/tasks/createTask.ts";
import { getTask } from "#src/services/tasks/getTask.ts";

export const taskRoutes = Router();

taskRoutes.get("/:taskId", async (req, res) => {
  const task = await getTask(Number(req.params.taskId));
  res.json(task);
});

taskRoutes.post("/", async (req, res) => {
  const newTask = NewTask.parse(req.body);
  const task = await createTask(newTask);
  res.status(201).json(task);
});
```

---

## Service

`src/services/tasks/getTask.ts`

```ts
import { NotFoundError } from "#src/errors/NotFoundError.ts";
import { selectTaskById } from "#src/queries/tasks/selectTaskById.ts";

export async function getTask(taskId: number) {
  const task = await selectTaskById(taskId);
  if (!task) {
    throw new NotFoundError(`Task ${taskId} not found`);
  }
  return task;
}
```

`src/services/tasks/createTask.ts`

```ts
import { insertTask } from "#src/queries/tasks/insertTask.ts";
import type { NewTask } from "#src/schemas/tasks/NewTask.ts";

export async function createTask(newTask: NewTask) {
  return insertTask(newTask);
}
```

`createTask` only calls one query today. It still exists, so every route
takes the same path, and the next rule for creating a task has a place to go.

---

## Query

`src/queries/tasks/selectTaskById.ts`

```ts
import { pool } from "#src/database/pool.ts";
import { Task } from "#src/schemas/tasks/Task.ts";

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

Every row goes through the zod schema, so the layers above get data they can
trust.

---

## Errors

`src/errors/NotFoundError.ts`

```ts
export class NotFoundError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "NotFoundError";
  }
}
```

`src/middleware/handleError.ts`

```ts
import type { ErrorRequestHandler } from "express";

const STATUS_BY_ERROR_NAME = new Map([
  ["ZodError", 400],
  ["NotAllowedError", 403],
  ["NotFoundError", 404],
]);

export const handleError: ErrorRequestHandler = (error, _req, res, _next) => {
  const status = STATUS_BY_ERROR_NAME.get(error.name);
  if (!status) {
    console.error(error);
    res.status(500).json({ message: "Something went wrong" });
    return;
  }
  res.status(status).json({ message: error.message });
};
```

A new kind of error is one new line in the map. The handler must take four
arguments, or Express does not see it as an error handler.

---

## Middleware

`src/middleware/checkSignedIn.ts`

```ts
import type { RequestHandler } from "express";

export const checkSignedIn: RequestHandler = (req, res, next) => {
  if (!req.headers.authorization) {
    res.status(401).json({ message: "Sign in first" });
    return;
  }
  next();
};
```

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
    route → service → query               route → query
    NewTask.parse(req.body)               createTask(req.body)
    throw new NotFoundError(...)          res.status(404) inside a service
    let handleError reply                 try { ... } catch in a route
    WHERE id = $1                         `WHERE id = ${taskId}`
    services/tasks/createTask.ts          services/taskService.ts with 20
                                          functions
    handler in taskRoutes.ts              a controllers/ folder
    getTask(taskId)                       getTask(req)
    app.use(handleError) last             app.use(handleError) first

---

## Before you commit

- Does every request go middleware → route → service → query, with no layer
  skipped or added?
- Is every route about fifteen lines or less, with its body checked by zod?
- Does any service use `req`, `res` or a status code?
- Does any query make a decision?
- Is there a `try` and `catch` in a route or service?
- Is every query parameterised, with no template-string values?
- Does every new error name have a line in `handleError`?
