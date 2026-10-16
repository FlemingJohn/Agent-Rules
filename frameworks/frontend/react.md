# Writing React

Builds on `core/` and `languages/typescript.md`. This file covers only what
is special to React. For a Next.js app, read `frameworks/frontend/nextjs.md`
as well.

Written for React 19.2.

---

## The rules

1. **One component per file.** A function component, a named export, and a
   file named after it.
2. **Props have their own type, in their own file.** `TaskCardProps.ts` holds
   `TaskCardProps`. Never a type sitting beside the component.
3. **Components show, hooks do.** A component reads values and draws them.
   Loading data, keeping state that many parts share and talking to the
   server go in a hook.
4. **Work values out, do not store them.** If a value can be worked out from
   props or state, work it out while drawing. Never copy it into its own
   state.
5. **Effects are for the outside world only.** `useEffect` talks to things
   outside React, such as a timer or a browser API. Never use it to work out
   a value or to react to a click.
6. **Keep state close.** State lives in the lowest component that needs it.
   Move it up only when a parent really needs it. Use context only when
   props would pass through more than two layers that do not use them.
7. **Lists use a real `key`.** The item's id. Never the array index.
8. **No `useMemo`, `useCallback` or `memo` by default.** Add one only when a
   real, measured slowness needs it, or turn on the React Compiler and let it
   do this for you.
9. **Colours, spacing and type come from the tokens.** Never write a colour
   by hand in a component.
10. **Return early for loading and errors.** The main view comes last and is
    never nested in a condition.

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Component | `TaskCard` | noun, `PascalCase` |
| Props type | `TaskCardProps` | component name, then `Props` |
| Hook | `useTasks` | `use` then what it gives |
| Event prop | `onClose` | `on` then what happened |
| Event handler | `handleClose` | `handle` then what happened |
| Yes or no prop | `isOpen`, `hasError` | starts with `is`, `has` or `can` |
| State pair | `[isOpen, setIsOpen]` | the value, then `set` and the value |
| Style file | `TaskCard.module.css` | component name, then `.module.css` |
| Context | `ThemeContext` | what it holds, then `Context` |

Never name a component for how it looks: `TaskCard`, not `BlueBox`.

---

## Folder layout

    src/
      main.tsx
      App.tsx
      styles/
        theme.css
      components/
        Button.tsx
        ButtonProps.ts
        Button.module.css
      tasks/
        Task.ts
        getTasks.ts
        useTasks.ts
        TaskList.tsx
        TaskCard.tsx
        TaskCardProps.ts
        TaskCard.module.css

`components/` holds plain building blocks used everywhere, such as buttons
and dialogs. Everything that belongs to one area of the product lives in that
area's folder.

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

`src/tasks/TaskCard.module.css`

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

## A hook that loads data

In a plain React app, load server data with TanStack Query. It handles
loading, errors, caching and refetching, which are easy to get wrong by hand.

`src/tasks/useTasks.ts`

```ts
import { useQuery } from "@tanstack/react-query";
import { getTasks } from "./getTasks";

export function useTasks() {
  return useQuery({ queryKey: ["tasks"], queryFn: getTasks });
}
```

---

## A component that uses it

`src/tasks/TaskList.tsx`

```tsx
import { TaskCard } from "./TaskCard";
import { useTasks } from "./useTasks";

export function TaskList() {
  const { data: tasks, isPending, error } = useTasks();
  if (isPending) {
    return <p>Loading tasks</p>;
  }
  if (error) {
    return <p>Could not load tasks</p>;
  }
  return (
    <ul>
      {tasks.map((task) => (
        <TaskCard key={task.id} task={task} />
      ))}
    </ul>
  );
}
```

---

## A component with props

`src/tasks/TaskCardProps.ts`

```ts
import type { Task } from "./Task";

export type TaskCardProps = {
  task: Task;
};
```

`src/tasks/TaskCard.tsx`

```tsx
import styles from "./TaskCard.module.css";
import type { TaskCardProps } from "./TaskCardProps";

export function TaskCard({ task }: TaskCardProps) {
  const isDone = task.status === "done";
  return (
    <li className={styles.card}>
      {task.title}
      {isDone && <span>Done</span>}
    </li>
  );
}
```

`isDone` is worked out while drawing. It is never its own state.

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
    export function TaskCard              export default function Card
    key={task.id}                         key={index}
    {tasks.length > 0 && <TaskList />}    {tasks.length && <TaskList />}
    var(--colour-accent)                  #2459d6 in a component
    onClose / handleClose                 closeCallback / doClose
    isOpen                                open, opened, showFlag
    a value worked out while drawing      a value copied into state
    plain function                        useCallback around everything
    if (isPending) return ...             a ternary three levels deep in JSX

---

## Before you commit

- Is there one component per file, with its props type in its own file?
- Does any component load data or hold shared state itself, instead of
  through a hook?
- Is any state a copy of something that could be worked out?
- Is any `useEffect` working out a value or reacting to a click?
- Does every list use the item's id as its `key`?
- Is there a `useMemo`, `useCallback` or `memo` with no measured reason?
- Is there a colour, size or font written by hand outside `theme.css`?
