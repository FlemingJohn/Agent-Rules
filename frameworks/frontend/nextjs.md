# Writing Next.js

Builds on `core/`, `languages/typescript.md` and
`frameworks/frontend/react.md`. This file covers only what is special to
Next.js.

Written for Next.js 16 with the App Router and plain SQL. The examples use
PostgreSQL through `pg`. For MySQL, see `databases/mysql.md`.

---

## The rules

1. **App Router only.** No `pages/` folder.
2. **`app/` holds routes only.** A `page.tsx` reads the data and hands it to
   a component from an area folder. The real components live outside `app/`.
3. **Server Components by default.** Add `"use client"` only to the smallest
   component that needs state, effects or browser events. Never to a page or
   a layout.
4. **Read data in Server Components.** A page calls the query function
   directly. No API route between your own page and your own database.
5. **Change data with Server Actions.** One action per file, `"use server"`
   at the top, input checked with zod, then `revalidatePath`.
6. **`route.ts` only for outside callers.** Webhooks, mobile apps, other
   services. Never for your own pages.
7. **Request values are awaited.** `await props.params`,
   `await props.searchParams`, `await cookies()`, `await headers()`.
8. **Server code says so.** Any file that touches the database or a secret
   starts with `import "server-only"`.
9. **No caching until it is needed.** Next.js 16 runs everything fresh by
   default. Turn on `cacheComponents` and add `"use cache"` only when a page
   is measured to be slow.
10. **`proxy.ts`, not `middleware.ts`.** The old name is deprecated.

---

## Names Next.js decides

These file names and default exports are fixed by Next.js. They are the only
exceptions to "name the file after the thing" and "named exports only":

| File | What it is | Export |
|---|---|---|
| `page.tsx` | a page | `export default` |
| `layout.tsx` | the frame around pages | `export default` |
| `loading.tsx` | shown while a page loads | `export default` |
| `error.tsx` | shown when a page fails | `export default` |
| `not-found.tsx` | shown for a missing page | `export default` |
| `route.ts` | an endpoint for outside callers | named `GET`, `POST` |
| `proxy.ts` | runs before every request | `export default` |

The function inside still gets a real name: `TasksPage`, `RootLayout`.

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Page component | `TasksPage`, `TaskPage` | what it shows, then `Page` |
| Layout component | `RootLayout` | where it sits, then `Layout` |
| Server Action | `createTaskAction` | verb and noun, then `Action` |
| Query function | `selectTasks`, `insertTask` | starts with the SQL verb |
| Route folder | `tasks/[taskId]` | plural nouns, ids in brackets |

---

## Folder layout

    src/
      app/
        layout.tsx
        page.tsx
        tasks/
          page.tsx
          [taskId]/
            page.tsx
      tasks/
        Task.ts
        NewTask.ts
        selectTasks.ts
        selectTaskById.ts
        insertTask.ts
        createTaskAction.ts
        TaskList.tsx
        TaskListProps.ts
        TaskDetails.tsx
        TaskDetailsProps.ts
        NewTaskForm.tsx
      database/
        pool.ts
      components/
        Button.tsx
      styles/
        theme.css
      settings.ts
      proxy.ts

Import with the `@/` alias, which points at `src/`:
`import { TaskList } from "@/tasks/TaskList"`.

---

## A page that reads data

`src/app/tasks/page.tsx`

```tsx
import { NewTaskForm } from "@/tasks/NewTaskForm";
import { selectTasks } from "@/tasks/selectTasks";
import { TaskList } from "@/tasks/TaskList";

export default async function TasksPage() {
  const tasks = await selectTasks();
  return (
    <main>
      <NewTaskForm />
      <TaskList tasks={tasks} />
    </main>
  );
}
```

---

## A page with an id in the path

`src/app/tasks/[taskId]/page.tsx`

```tsx
import { notFound } from "next/navigation";
import { selectTaskById } from "@/tasks/selectTaskById";
import { TaskDetails } from "@/tasks/TaskDetails";

export default async function TaskPage(props: PageProps<"/tasks/[taskId]">) {
  const { taskId } = await props.params;
  const task = await selectTaskById(Number(taskId));
  if (!task) {
    notFound();
  }
  return <TaskDetails task={task} />;
}
```

`PageProps` is made by Next.js from your folders, so the page needs no props
file of its own.

---

## Server-only code

`src/database/pool.ts`

```ts
import "server-only";
import pg from "pg";
import { settings } from "@/settings";

export const pool = new pg.Pool({ connectionString: settings.databaseUrl });
```

`src/tasks/selectTasks.ts`

```ts
import "server-only";
import { pool } from "@/database/pool";
import { Task } from "./Task";

const SELECT_TASKS = `
  SELECT id, title, status, created_at AS "createdAt"
  FROM tasks
  ORDER BY created_at DESC
`;

export async function selectTasks() {
  const result = await pool.query(SELECT_TASKS);
  return Task.array().parse(result.rows);
}
```

If a client component ever imports these files, the build fails instead of
sending database code to the browser.

---

## A Server Action

`src/tasks/createTaskAction.ts`

```ts
"use server";

import { revalidatePath } from "next/cache";
import { insertTask } from "./insertTask";
import { NewTask } from "./NewTask";

export async function createTaskAction(formData: FormData) {
  const newTask = NewTask.parse({ title: formData.get("title") });
  await insertTask(newTask);
  revalidatePath("/tasks");
}
```

`src/tasks/NewTaskForm.tsx`

```tsx
import { createTaskAction } from "./createTaskAction";

export function NewTaskForm() {
  return (
    <form action={createTaskAction}>
      <input name="title" required maxLength={200} />
      <button type="submit">Add task</button>
    </form>
  );
}
```

The form works with no client JavaScript at all. Add `"use client"` and
`useActionState` only when the form needs to show a pending state or an
error message.

---

## Settings

`src/settings.ts`

```ts
import "server-only";
import { z } from "zod";

const Settings = z.object({
  DATABASE_URL: z.string(),
});

const environment = Settings.parse(process.env);

export const settings = {
  databaseUrl: environment.DATABASE_URL,
};
```

Only values that are safe for anyone to see start with `NEXT_PUBLIC_`.
Everything else stays on the server.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    page calls selectTasks()              page fetches /api/tasks
    createTaskAction in its own file      a route.ts for your own form
    "use client" on a small button        "use client" on a whole page
    const { taskId } = await props.params props.params.taskId
    await cookies()                       cookies().get(...)
    import "server-only" in pool.ts       database code with no guard
    proxy.ts                              middleware.ts
    NEXT_PUBLIC_ only for safe values     NEXT_PUBLIC_DATABASE_URL
    no caching until measured             "use cache" on everything
    components in src/tasks/              components inside src/app/

---

## Before you commit

- Does `app/` hold only route files, with real components in area folders?
- Is `"use client"` on a page, a layout or a large component?
- Does any page fetch its own API instead of calling the query directly?
- Is every change to data a Server Action that checks its input with zod?
- Are `params`, `searchParams`, `cookies()` and `headers()` all awaited?
- Does every file that touches the database start with
  `import "server-only"`?
- Is there a secret in a `NEXT_PUBLIC_` variable?
