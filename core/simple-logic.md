# Simple logic

Code should be easy to follow for someone who did not write it, including an
agent opening it cold. Plain and a little longer beats clever and short.

---

## The rules

1. **Return early.** Handle the bad cases first and return. The main path is
   never nested.
2. **No helper until it has two callers.** A function used once stays inline,
   unless the main function is too long without it.
3. **No layers beyond the standard ones.** A backend has middleware, routes,
   services and queries. A frontend has pages, components, hooks and
   services. Never add a manager, wrapper or extra layer between them.
4. **No code for later.** No options, flags, base classes or settings for
   something nobody needs yet.
5. **Look up, do not branch.** Three or more `if` or `else` arms that pick a
   value become a map.
6. **One step per line.** No chained one-liners, nested ternaries or clever
   tricks. Name the middle value.
7. **Use what the language gives you.** No new library for something the
   language or framework already does.
8. **Let errors through.** Do not catch an error you cannot handle. Never
   catch and ignore.

---

## Return early

    good                                  bad
    ──────────────────────────────────────────────────────────────
    if task is None:                      if task is not None:
        raise TaskNotFound(task_id)           if task.owner_id == user.id:
    if task.owner_id != user.id:                  return task
        raise NotAllowed()                    else:
    return task                                   raise NotAllowed()
                                          else:
                                              raise TaskNotFound(task_id)

---

## No helper until it has two callers

    good                                  bad
    ──────────────────────────────────────────────────────────────
    def create_task(new_task):            def create_task(new_task):
        title = new_task.title.strip()        title = _get_clean_title(new_task)
        ...                                   ...

                                          def _get_clean_title(new_task):
                                              return new_task.title.strip()

A one-line helper used once makes the reader jump for nothing.

---

## No layers beyond the standard ones

    good                                  bad
    ──────────────────────────────────────────────────────────────
    route → service → query               route → controller → manager
                                            → service → repository
                                            → query

A service that only calls one query today is fine. Every request taking the
same path through the same layers matters more than saving one file. What is
not fine is a layer the framework file does not list.

---

## Look up, do not branch

    good                                  bad
    ──────────────────────────────────────────────────────────────
    COLOUR_BY_STATUS = {                  if status == "open":
        "open": "blue",                       colour = "blue"
        "done": "green",                  elif status == "done":
        "late": "red",                        colour = "green"
    }                                     elif status == "late":
    colour = COLOUR_BY_STATUS[status]         colour = "red"

A new status is one new line, not a new branch.

---

## One step per line

Good:

    open_tasks = [task for task in tasks if task.is_open]
    owners = {task.owner for task in open_tasks}
    owner_names = sorted(owner.name for owner in owners)

Bad:

    names = ",".join(sorted({t.owner.name.title() for t in tasks if t.is_open and t.owner}))

Good:

    label = "late" if is_late else "on time"

Bad:

    label = "late" if is_late else "soon" if is_close else "on time"

One ternary is fine. A ternary inside a ternary is not.

---

## Let errors through

    good                                  bad
    ──────────────────────────────────────────────────────────────
    task = select_task(connection,        try:
                       task_id)               task = select_task(connection, task_id)
                                          except Exception:
                                              pass

Catch an error only where you can do something useful with it, such as
turning it into a clear message for the user.

---

## Before you commit

- Is any `if` nested inside another `if` where an early return would do?
- Is there a helper with only one caller that could sit inline?
- Is there a layer that the framework file does not list?
- Is there an option, flag or setting nobody uses yet?
- Are there three or more branches that pick a value?
- Is there a line you had to read twice?
- Is there a `try` that catches everything, or catches and does nothing?
