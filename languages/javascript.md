# Writing JavaScript

Builds on `core/code-shape.md`, `core/naming.md` and `core/simple-logic.md`.
This file covers only what is special to JavaScript.

For a new project, use TypeScript (`languages/typescript.md`). Use this file
when the project is already JavaScript or a tool demands plain `.js`.

Written for modern JavaScript (ES2024 and newer) as run by Node 24 and
current browsers.

---

## The rules

1. **ES modules only.** `import` and `export`. Never `require` or
   `module.exports`. Set `"type": "module"` in `package.json`.
2. **Check outside data with zod.** Request bodies, form data, API replies
   and environment variables go through a zod schema before the program uses
   them. With no types, this is the only thing that keeps the data honest.
3. **Named exports only.** No `export default`, except where a framework
   demands it.
4. **`const` by default.** `let` only when the value really changes. Never
   `var`.
5. **`===` and `!==` only.** Never `==` or `!=`.
6. **`async` and `await`.** Never `.then` chains, never callbacks for things
   that can be awaited.
7. **Throw a named error.** Your own error class, in its own file.
8. **No JSDoc.** No type comments. If the code needs types to be clear, move
   the project to TypeScript.
9. **Never change what you were given.** Return a new object or array.

---

## Tools

| Job | Tool | Command |
|---|---|---|
| Lint and format | ESLint and Prettier, or Biome | whatever the project already uses |
| Tests | Vitest, or the Node test runner | `npm test` |

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Function | `createTask` | verb then noun, `camelCase` |
| Yes or no function | `canEditTask` | starts with `is`, `has` or `can` |
| Zod schema | `Task` | noun, `PascalCase` |
| Class | `TaskNotFoundError` | noun, `PascalCase` |
| Constant | `MAX_TITLE_LENGTH` | `UPPER_SNAKE`, at the top of the file |
| File | `createTask.js`, `Task.js` | the thing's name, in the thing's case |
| Test file | `createTask.test.js` | the file it tests, then `.test` |

---

## Folder layout

Folders are layers, and the framework file names them. Inside a layer, one
folder per area. One thing per file.

    src/
      main.js
      config/
        settings.js
      schemas/
        tasks/
          Task.js
          TaskStatus.js
      errors/
        TaskNotFoundError.js
      services/
        tasks/
          closeTask.js
          closeTask.test.js

---

## Checking outside data

`src/schemas/tasks/Task.js`

```js
import { z } from "zod";
import { TaskStatus } from "./TaskStatus.js";

export const Task = z.object({
  id: z.number(),
  title: z.string(),
  status: TaskStatus,
});
```

```js
const task = Task.parse(body);
```

After `parse`, the program knows `task` has the right shape. Before it, it
knows nothing.

---

## A function

`src/services/tasks/closeTask.js`

```js
export function closeTask(task) {
  if (task.status === "done") {
    return task;
  }
  return { ...task, status: "done" };
}
```

---

## Imports

Write the file ending in every import: `./Task.js`, not `./Task`. Node needs
it for ES modules.

Import across folders from the `#src/` path set up in
`frameworks/backend/node.md`: `#src/schemas/tasks/Task.js`, never
`../../schemas/tasks/Task.js`. Files in the same folder are imported with
`./`.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    import { Task } from "./Task.js"      const Task = require("./Task")
    export function closeTask             module.exports = closeTask
    const total = 0                       var total = 0
    task.status === "done"                task.status == "done"
    const tasks = await getTasks()        getTasks().then(tasks => ...)
    return { ...task, status: "done" }    task.status = "done"; return task
    const task = Task.parse(body)         const task = body
    throw new TaskNotFoundError(id)       throw "not found"
    const tasks = [...list, newTask]      list.push(newTask)

---

## Before you commit

- Is there a `require`, `module.exports` or `var`?
- Does every piece of outside data go through a zod schema?
- Is there a `==`, `!=` or `.then` chain?
- Does any function change an object or array it was given?
- Does every import end in `.js`?
- Is there a JSDoc block? Remove it.
- Do the linter and the tests pass?
