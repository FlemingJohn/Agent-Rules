# Writing React

Builds on `core/` and `languages/typescript.md`. This file covers only what
is special to React. For a Next.js app, read `frameworks/frontend/nextjs.md`
as well.

Written for React 19.2.

---

## The layers

| Folder | Its one job | Never |
|---|---|---|
| `pages/` | one screen: loads what it needs and lays out components | small reusable pieces |
| `components/` | draw something on screen, one folder per component | call the server directly |
| `hooks/` | state and loading shared by more than one component | draw anything |
| `services/` | talk to the backend: one request per file | React code |
| `types/` | shapes used in more than one place | logic |
| `styles/` | the design tokens: colours, spacing, type | component styles |

---

## The rules

1. **One folder per component.** `components/tasks/TaskCard/` holds
   everything `TaskCard` needs: the component, its styles, its props type,
   any hook only it uses, and its test.
2. **Used by one, stays in the folder. Used by two, moves up.** A hook or
   type that a second component needs moves to `hooks/` or `types/`.
3. **Group component folders.** `components/ui/` for plain building blocks
   such as buttons and dialogs. `components/<area>/` for each area of the
   product, such as `components/tasks/`.
4. **No `index.ts` files.** Import the file itself:
   `@/components/tasks/TaskCard/TaskCard`.
5. **Components show, hooks do.** A component reads values and draws them.
   Loading data and shared state go in a hook.
6. **Work values out, do not store them.** If a value can be worked out from
   props or state, work it out while drawing. Never copy it into its own
   state.
7. **Effects are for the outside world only.** `useEffect` talks to things
   outside React, such as a timer or a browser API. Never use it to work out
   a value or to react to a click.
8. **Keep state close.** State lives in the lowest component that needs it.
   Use context only when props would pass through more than two layers that
   do not use them.
9. **Lists use a real `key`.** The item's id. Never the array index.
10. **No `useMemo`, `useCallback` or `memo` by default.** Add one only for a
    real, measured slowness, or turn on the React Compiler and let it do this
    for you.
11. **Colours, spacing and type come from the tokens.** Never write a colour
    by hand in a component.
12. **Return early for loading and errors.** The main view comes last and is
    never nested in a condition.

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Component folder and file | `TaskCard/TaskCard.tsx` | noun, `PascalCase`, same name twice |
| Props type | `TaskCardProps` | component name, then `Props` |
| Style file | `TaskCard.module.css` | component name, then `.module.css` |
| Test file | `TaskCard.test.tsx` | component name, then `.test` |
| Page | `TasksPage` | what it shows, then `Page` |
| Hook | `useTasks` | `use` then what it gives |
| Service | `getTasks`, `createTask` | verb then noun |
| Event prop | `onClose` | `on` then what happened |
| Event handler | `handleClose` | `handle` then what happened |
| Yes or no prop | `isOpen`, `hasError` | starts with `is`, `has` or `can` |
| State pair | `[isOpen, setIsOpen]` | the value, then `set` and the value |

Never name a component for how it looks: `TaskCard`, not `BlueBox`.

---

## Folder layout

    src/
      main.tsx
      App.tsx
      pages/
        TasksPage/
          TasksPage.tsx
          TasksPage.module.css
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
          TaskCard/
            TaskCard.tsx
            TaskCard.module.css
            TaskCardProps.ts
            useIsExpanded.ts
            TaskCard.test.tsx
      hooks/
        useTasks.ts
      services/
        getTasks.ts
      types/
        Task.ts
        TaskStatus.ts
      styles/
        theme.css

`useIsExpanded` sits in `TaskCard/` because only `TaskCard` uses it.
`useTasks` sits in `hooks/` because more than one component uses it.

Delete a component folder and nothing it used is left behind elsewhere.

---

## Tokens

`src/styles/theme.css`

```css
:root {
  --colour-text: #1a1a1a;
  --colour-surface: #ffffff;
  --colour-accent: #2459d6;
  --space-2: 8px;
  --space-3: 12px;
  --radius-2: 8px;
}
```

This is the only file with a colour written by hand. If a colour is not here,
it does not belong in the product.

`src/components/tasks/TaskCard/TaskCard.module.css`

```css
.card {
  padding: var(--space-3);
  background: var(--colour-surface);
  border-radius: var(--radius-2);
}
```

If the project uses Tailwind, the tokens go in the Tailwind theme and the
same rule holds: no raw colour values in class names.

---

## A service

`src/services/getTasks.ts`

```ts
import { Task } from "@/types/Task";

export async function getTasks() {
  const response = await fetch("/api/tasks");
  if (!response.ok) {
    throw new Error(`Could not load tasks: ${response.status}`);
  }
  return Task.array().parse(await response.json());
}
```

---

## A shared hook

In a plain React app, load server data with TanStack Query. It handles
loading, errors, caching and refetching, which are easy to get wrong by hand.

`src/hooks/useTasks.ts`

```ts
import { useQuery } from "@tanstack/react-query";
import { getTasks } from "@/services/getTasks";

export function useTasks() {
  return useQuery({ queryKey: ["tasks"], queryFn: getTasks });
}
```

---

## A component that uses it

`src/components/tasks/TaskList/TaskList.tsx`

```tsx
import { TaskCard } from "@/components/tasks/TaskCard/TaskCard";
import { useTasks } from "@/hooks/useTasks";
import styles from "./TaskList.module.css";

export function TaskList() {
  const { data: tasks, isPending, error } = useTasks();
  if (isPending) {
    return <p>Loading tasks</p>;
  }
  if (error) {
    return <p>Could not load tasks</p>;
  }
  return (
    <ul className={styles.list}>
      {tasks.map((task) => (
        <TaskCard key={task.id} task={task} />
      ))}
    </ul>
  );
}
```

---

## A component with its own folder

`src/components/tasks/TaskCard/TaskCardProps.ts`

```ts
import type { Task } from "@/types/Task";

export type TaskCardProps = {
  task: Task;
};
```

`src/components/tasks/TaskCard/useIsExpanded.ts`

```ts
import { useState } from "react";

export function useIsExpanded() {
  const [isExpanded, setIsExpanded] = useState(false);
  function handleToggle() {
    setIsExpanded(!isExpanded);
  }
  return { isExpanded, handleToggle };
}
```

`src/components/tasks/TaskCard/TaskCard.tsx`

```tsx
import styles from "./TaskCard.module.css";
import type { TaskCardProps } from "./TaskCardProps";
import { useIsExpanded } from "./useIsExpanded";

export function TaskCard({ task }: TaskCardProps) {
  const { isExpanded, handleToggle } = useIsExpanded();
  const isDone = task.status === "done";
  return (
    <li className={styles.card}>
      <button type="button" onClick={handleToggle}>
        {task.title}
      </button>
      {isDone && <span>Done</span>}
      {isExpanded && <p>Created {task.createdAt.toLocaleDateString()}</p>}
    </li>
  );
}
```

Files inside the component's own folder are imported with `./`. Everything
outside it is imported with `@/`.

---

## Work values out, do not store them

    good                                  bad
    ──────────────────────────────────────────────────────────────
    const openTasks = tasks.filter(       const [openTasks, setOpenTasks] =
      (task) => task.status === "open"      useState([]);
    );                                    useEffect(() => {
                                            setOpenTasks(tasks.filter(...));
                                          }, [tasks]);

---

## Events

    good                                  bad
    ──────────────────────────────────────────────────────────────
    function handleClose() {              useEffect(() => {
      setIsOpen(false);                     if (wasClicked) setIsOpen(false);
      onClose();                          }, [wasClicked]);
    }
    <button onClick={handleClose}>

Things that happen because of a click go in the click handler.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    components/tasks/TaskCard/            components/TaskCard.tsx with its
      TaskCard.tsx, its css, its props      css in styles/ and props in
                                            types/
    a hook one component uses, in its     every hook in hooks/
    folder
    @/components/tasks/TaskCard/TaskCard  @/components/tasks/TaskCard via
                                          an index.ts
    getTasks() in services/               fetch() inside a component
    key={task.id}                         key={index}
    {tasks.length > 0 && <TaskList />}    {tasks.length && <TaskList />}
    var(--colour-accent)                  #2459d6 in a component
    a value worked out while drawing      a value copied into state
    plain function                        useCallback around everything

---

## Before you commit

- Does every component have its own folder holding its component, styles,
  props type and test?
- Is a hook or type that only one component uses sitting outside its folder?
  Is one that two components use still inside a component folder?
- Is there an `index.ts` anywhere?
- Does any component call `fetch` itself instead of through a service and
  hook?
- Is any state a copy of something that could be worked out?
- Is any `useEffect` working out a value or reacting to a click?
- Does every list use the item's id as its `key`?
- Is there a colour, size or font written by hand outside `theme.css`?
