# Writing TypeScript

Builds on `core/code-shape.md`, `core/naming.md` and `core/simple-logic.md`.
This file covers only what is special to TypeScript.

Written for TypeScript 7 with strict mode on.

---

## The rules

1. **Strict mode, always.** `"strict": true` and
   `"noUncheckedIndexedAccess": true` in `tsconfig.json`.
2. **No `any`.** Use `unknown` for data you have not checked yet, then check
   it.
3. **No casts.** No `as Task`, no `!` after a value. The only cast allowed is
   `as const`. If the types disagree, fix the types.
4. **Check outside data with zod.** Request bodies, form data, API replies,
   environment variables and database rows go through a zod schema. The type
   comes from the schema, so it is written once.
5. **`type`, not `interface`.** One way to write a shape, everywhere.
6. **A fixed set of values is a union of strings.** `"open" | "done"`. Never
   `enum`.
7. **Named exports only.** No `export default`, except where a framework
   demands it, such as a Next.js `page.tsx`.
8. **`const` by default.** `let` only when the value really changes. Never
   `var`.
9. **`async` and `await`.** Never `.then` chains.
10. **Throw a named error.** Your own error class, in its own file. Never
    `throw "message"`.

---

## Tools

| Job | Tool | Command |
|---|---|---|
| Types | TypeScript | `tsc --noEmit` |
| Lint and format | ESLint and Prettier, or Biome | whatever the project already uses |
| Tests | Vitest, or the Node test runner | `npm test` |

Run them all before every commit.

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Function | `createTask` | verb then noun, `camelCase` |
| Yes or no function | `canEditTask` | starts with `is`, `has` or `can` |
| Type | `Task` | noun, `PascalCase` |
| Zod schema | `Task` | same name as its type |
| Union of strings | `TaskStatus` | area then `Status` |
| Union values | `"open"`, `"done"` | lower case words |
| Error class | `TaskNotFoundError` | says what went wrong, ends with `Error` |
| Constant | `MAX_TITLE_LENGTH` | `UPPER_SNAKE`, at the top of the file |
| File | `createTask.ts`, `Task.ts` | the thing's name, in the thing's case |
| Test file | `createTask.test.ts` | the file it tests, then `.test` |

`createTask.ts` exports `createTask`. `Task.ts` exports `Task`.

---

## Folder layout

Folders are layers, and the framework file names them: `react.md`,
`nextjs.md` and `express.md` each give the full tree. Inside a layer, one
folder per area once there is more than one area. One thing per file.

    src/
      main.ts
      types/
        Task.ts
        TaskStatus.ts
        NewTask.ts
      errors/
        TaskNotFoundError.ts
      services/
        tasks/
          closeTask.ts
          closeTask.test.ts

No `types.ts`, `utils.ts` or `index.ts` files that gather other files
together. Import each thing from its own file.

Import from the root alias, not long relative paths: `@/types/Task` with a
bundler, as in Next.js, or `#src/types/Task.ts` when Node runs the files
directly. See `frameworks/backend/node.md`. Files in the same folder are
imported with `./`.

---

## A fixed set of values

`src/types/TaskStatus.ts`

```ts
import { z } from "zod";

export const TaskStatus = z.enum(["open", "done"]);
export type TaskStatus = z.infer<typeof TaskStatus>;
```

The schema and its type share one name. Together they are one thing.

---

## A shape

`src/types/Task.ts`

```ts
import { z } from "zod";
import { TaskStatus } from "./TaskStatus";

export const Task = z.object({
  id: z.number(),
  title: z.string(),
  status: TaskStatus,
  createdAt: z.coerce.date(),
});
export type Task = z.infer<typeof Task>;
```

`z.coerce.date()` accepts both a `Date` from the database and a date string
from JSON.

A shape that never comes from outside the program needs no schema:

```ts
export type TaskCount = {
  open: number;
  done: number;
};
```

---

## Errors

`src/errors/TaskNotFoundError.ts`

```ts
export class TaskNotFoundError extends Error {
  readonly taskId: number;

  constructor(taskId: number) {
    super(`Task ${taskId} not found`);
    this.taskId = taskId;
  }
}
```

Write fields out. Never the `constructor(public taskId: number)` shorthand.

---

## A function

`src/services/tasks/closeTask.ts`

```ts
import type { Task } from "@/types/Task";

export function closeTask(task: Task): Task {
  if (task.status === "done") {
    return task;
  }
  return { ...task, status: "done" };
}
```

Never change an object that was passed in. Return a new one.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    const body: unknown = ...             const body: any = ...
    const task = Task.parse(body)         const task = body as Task
    if (!task) return                     task!.title
    type Task = { ... }                   interface Task { ... }
    type Status = "open" | "done"         enum Status { Open, Done }
    export function closeTask             export default function
    const total = 0                       var total = 0
    const tasks = await getTasks()        getTasks().then(tasks => ...)
    throw new TaskNotFoundError(id)       throw "not found"
    import type { Task } from             import { Task } from "./types"
      "@/types/Task"
    task.status === "done"                task.status == "done"

---

## Before you commit

- Is `any`, `as` (other than `as const`) or `!` used anywhere?
- Does every piece of outside data go through a zod schema?
- Is every shape a `type` and every fixed set of values a string union?
- Is there a default export where the framework does not demand one?
- Is there a `.then` chain or a `var`?
- Is every error a named class in its own file?
- Do `tsc --noEmit`, the linter and the tests all pass?
