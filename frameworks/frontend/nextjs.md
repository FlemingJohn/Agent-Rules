# Writing Next.js

Builds on `core/`, `languages/typescript.md` and
`frameworks/frontend/react.md`. This file covers only what is special to
Next.js.

Written for Next.js 16 with the App Router and plain SQL. The examples use
PostgreSQL through `pg`. For MySQL, see `databases/mysql.md`.

---

## The layers

Next.js runs code in two places: the server and the browser. The folders
keep the two apart.

    page or form → action → service → query → database

| Folder | Its one job | Runs on |
|---|---|---|
| `app/` | routes only: one folder per web address | server |
| `components/` | draw something, one folder per component, as in `react.md` | server, or browser if `"use client"` |
| `hooks/` | state shared by more than one client component | browser |
| `types/` | shapes used in more than one place | both |
| `server/actions/` | Server Actions: what a form calls to change data | server |
| `server/services/` | the business rules | server |
| `server/queries/` | one SQL statement each | server |
| `server/database/` | the database connection | server |
| `server/config/` | settings read from the environment | server |
| `styles/` | the design tokens | both |

Everything under `server/` must never reach the browser. Each file there
starts with `import "server-only"`, so the build fails if it is ever imported
into client code.

---

## The rules

1. **App Router only.** No `pages/` folder.
2. **`app/` holds routes only.** A `page.tsx` calls a service and hands the
   result to a component. The real components live in `components/`.
3. **Components follow `react.md`.** One folder per component, grouped by
   area, holding its styles, props type, private hook and test.
4. **Server Components by default.** Add `"use client"` only to the smallest
   component that needs state, effects or browser events. Never to a page or
   a layout.
5. **Read data by calling a service.** A page calls
   `server/services/...` directly. No API route between your own page and
   your own database.
6. **Change data with a Server Action.** One action per file in
   `server/actions/`. It checks the form with zod, calls one service, then
   calls `revalidatePath`.
7. **`route.ts` only for outside callers.** Webhooks, mobile apps, other
   services. Never for your own pages.
8. **Request values are awaited.** `await props.params`,
   `await props.searchParams`, `await cookies()`, `await headers()`.
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
| Server Action | `createTaskAction` | the service it calls, then `Action` |
| Service | `getTasks`, `createTask` | verb then noun |
| Query | `selectTasks`, `insertTask` | starts with the SQL verb |
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
      components/
        ui/
          Button/
            Button.tsx
            Button.module.css
            ButtonProps.ts
        tasks/
          TaskList/
            TaskList.tsx
            TaskList.module.css
            TaskListProps.ts
          TaskDetails/
            TaskDetails.tsx
            TaskDetails.module.css
            TaskDetailsProps.ts
          NewTaskForm/
            NewTaskForm.tsx
            NewTaskForm.module.css
      hooks/
      types/
        Task.ts
        NewTask.ts
        TaskStatus.ts
      server/
        actions/
          tasks/
            createTaskAction.ts
        services/
          tasks/
            getTasks.ts
            getTask.ts
            createTask.ts
        queries/
          tasks/
            selectTasks.ts
            selectTaskById.ts
            insertTask.ts
        database/
          pool.ts
        config/
          settings.ts
      styles/
        theme.css
      proxy.ts

Import with the `@/` alias, which points at `src/`. No `index.ts` files.

---

## A page that reads data

`src/app/tasks/page.tsx`

```tsx
import { NewTaskForm } from "@/components/tasks/NewTaskForm/NewTaskForm";
import { TaskList } from "@/components/tasks/TaskList/TaskList";
import { getTasks } from "@/server/services/tasks/getTasks";

export default async function TasksPage() {
  const tasks = await getTasks();
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
import { TaskDetails } from "@/components/tasks/TaskDetails/TaskDetails";
import { getTask } from "@/server/services/tasks/getTask";

export default async function TaskPage(props: PageProps<"/tasks/[taskId]">) {
  const { taskId } = await props.params;
  const task = await getTask(Number(taskId));
  if (!task) {
    notFound();
  }
  return <TaskDetails task={task} />;
}
```

`PageProps` is made by Next.js from your folders, so the page needs no props
file of its own.

---

## Service and query

`src/server/services/tasks/getTasks.ts`

```ts
import "server-only";
import { selectTasks } from "@/server/queries/tasks/selectTasks";

export async function getTasks() {
  return selectTasks();
}
```

`src/server/queries/tasks/selectTasks.ts`

```ts
import "server-only";
import { pool } from "@/server/database/pool";
import { Task } from "@/types/Task";

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

`getTasks` only calls one query today. It still exists, so every page takes
the same path, and the next rule has a place to go.

---

## The database and settings

`src/server/database/pool.ts`

```ts
import "server-only";
import pg from "pg";
import { settings } from "@/server/config/settings";

export const pool = new pg.Pool({ connectionString: settings.databaseUrl });
```

`src/server/config/settings.ts`

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

## A Server Action

`src/server/actions/tasks/createTaskAction.ts`

```ts
"use server";

import { revalidatePath } from "next/cache";
import { createTask } from "@/server/services/tasks/createTask";
import { NewTask } from "@/types/NewTask";

export async function createTaskAction(formData: FormData) {
  const newTask = NewTask.parse({ title: formData.get("title") });
  await createTask(newTask);
  revalidatePath("/tasks");
}
```

`src/components/tasks/NewTaskForm/NewTaskForm.tsx`

```tsx
import { createTaskAction } from "@/server/actions/tasks/createTaskAction";
import styles from "./NewTaskForm.module.css";

export function NewTaskForm() {
  return (
    <form className={styles.form} action={createTaskAction}>
      <input name="title" required maxLength={200} />
      <button type="submit">Add task</button>
    </form>
  );
}
```

A component may import a Server Action, because `"use server"` turns it into
a safe call to the server. It must never import a service or query.

The form works with no client JavaScript at all. Add `"use client"` and
`useActionState` only when the form needs to show a pending state or an
error message.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    page → service → query                page → query
    page calls getTasks()                 page fetches /api/tasks
    createTaskAction in server/actions/   a route.ts for your own form
    component imports an action           component imports a service
    "use client" on a small button        "use client" on a whole page
    const { taskId } = await props.params props.params.taskId
    await cookies()                       cookies().get(...)
    import "server-only" in server/ files server code with no guard
    proxy.ts                              middleware.ts
    NEXT_PUBLIC_ only for safe values     NEXT_PUBLIC_DATABASE_URL
    components/tasks/TaskList/            components inside src/app/

---

## Before you commit

- Does `app/` hold only route files, with real components in `components/`?
- Does every page and action go through a service, never straight to a
  query?
- Does every file under `server/` start with `import "server-only"`, apart
  from actions, which start with `"use server"`?
- Does any component import from `server/services` or `server/queries`?
- Is `"use client"` on a page, a layout or a large component?
- Are `params`, `searchParams`, `cookies()` and `headers()` all awaited?
- Is there a secret in a `NEXT_PUBLIC_` variable?
