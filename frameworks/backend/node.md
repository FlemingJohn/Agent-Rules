# Writing Node.js

Builds on `core/` and `languages/typescript.md` or
`languages/javascript.md`. This file covers only what is special to Node.

Written for Node 24 LTS. Node 26 becomes LTS on 28 October 2026, and
nothing here changes with it.

---

## The rules

1. **ES modules.** `"type": "module"` in `package.json`.
2. **Built-in modules use the `node:` prefix.** `node:fs/promises`,
   `node:path`, `node:crypto`.
3. **Use what Node already has.** Before adding a package, check the list
   below.
4. **Settings are read once, in one file.** `config/settings.ts` reads
   `process.env`, checks it with zod and exports the result. Nothing else
   touches `process.env`.
5. **Promises, never callbacks.** `node:fs/promises`, not `node:fs` with
   callbacks.
6. **Never block the server.** No `readFileSync` or other `Sync` calls inside
   a request. They are fine at start-up.
7. **Fail at start-up, not later.** A missing setting or a database that
   cannot be reached stops the app before it starts listening.
8. **Building the app and starting it are two files.** One file makes the
   app, so tests can use it. `main.ts` starts it.
9. **Import from `#src/`, not long relative paths.**
   `#src/services/tasks/closeTask.ts`, never `../../services/tasks/closeTask.ts`.

---

## What Node already has

| Need | Use | Not |
|---|---|---|
| Read `.env` | `node --env-file=.env` | `dotenv` |
| Restart on change | `node --watch` | `nodemon` |
| Run TypeScript | `node src/main.ts` | `ts-node` |
| HTTP requests | `fetch` | `axios`, `node-fetch` |
| Tests | `node --test` | a test library, for small projects |
| Random ids | `crypto.randomUUID()` | `uuid` |

Node 24 runs `.ts` files directly by removing the types. `tsc --noEmit`
still checks them.

---

## TypeScript set-up for Node

`tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "allowImportingTsExtensions": true,
    "erasableSyntaxOnly": true,
    "verbatimModuleSyntax": true
  }
}
```

Because Node runs the files as they are:

- Every import ends in `.ts`: `import { Task } from "./Task.ts"`.
- Type-only imports say so: `import type { Task } from "./Task.ts"`.
- No `enum`, no `namespace`, no constructor shorthand like
  `constructor(public taskId: number)`. Write the field out instead.

---

## Scripts

`package.json`

```json
{
  "type": "module",
  "imports": {
    "#src/*": "./src/*"
  },
  "scripts": {
    "dev": "node --env-file=.env --watch src/main.ts",
    "start": "node --env-file=.env src/main.ts",
    "check": "tsc --noEmit",
    "test": "node --test"
  }
}
```

Every project has the same four names, so anyone can run it.

`"imports"` lets every file import with `#src/` instead of counting `../`.
Node and TypeScript both understand it with no extra package.

---

## Folder layout

Folders are layers. Inside each layer, one folder per area. A server built
on Node follows the layers in `frameworks/backend/express.md`.

    project/
      package.json
      tsconfig.json
      .env
      src/
        main.ts
        config/
          settings.ts
        services/
          tasks/
            closeTask.ts
            closeTask.test.ts
        schemas/
          tasks/
            Task.ts
        errors/
          NotFoundError.ts

A test sits next to the file it tests.

---

## Settings

`src/config/settings.ts`

```ts
import { z } from "zod";

const Settings = z.object({
  DATABASE_URL: z.string(),
  PORT: z.coerce.number().default(3000),
});

const environment = Settings.parse(process.env);

export const settings = {
  databaseUrl: environment.DATABASE_URL,
  port: environment.PORT,
};
```

If a setting is missing, `parse` throws at start-up with the name of the
missing setting.

---

## A test with the Node test runner

`src/services/tasks/closeTask.test.ts`

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import { closeTask } from "#src/services/tasks/closeTask.ts";

test("closeTask marks the task done", () => {
  const task = { id: 1, title: "Write plan", status: "open" as const, createdAt: new Date() };
  assert.equal(closeTask(task).status, "done");
});
```

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    import { readFile } from              const fs = require("fs")
      "node:fs/promises"
    await readFile(path, "utf8")          readFileSync(path) inside a request
    settings.databaseUrl                  process.env.DATABASE_URL in a route
    node --env-file=.env                  import "dotenv/config"
    node --watch                          nodemon
    fetch(url)                            axios.get(url)
    import { Task } from "./Task.ts"      import { Task } from "./Task"
    "#src/schemas/tasks/Task.ts"          "../../../schemas/tasks/Task.ts"

---

## Before you commit

- Is `process.env` read anywhere other than `config/settings.ts`?
- Is there an import that climbs with `../../` instead of using `#src/`?
- Is there a `Sync` call that runs during a request?
- Did you add a package for something Node already does?
- Does every TypeScript import end in `.ts`?
- Do `npm run check` and `npm test` pass?
