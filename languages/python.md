# Writing Python

Builds on `core/code-shape.md`, `core/naming.md` and `core/simple-logic.md`.
This file covers only what is special to Python.

Written for Python 3.12 or newer. The latest release is 3.15.

---

## The rules

1. **Type every function.** Every argument and every return value has a type.
2. **Modern type syntax.** `list[str]`, `dict[str, int]`, `str | None`. Never
   `List`, `Dict` or `Optional` from `typing`.
3. **Pick the shape by where the data is.** Pydantic model at the edges, where
   data comes in from outside. Dataclass inside the program. `TypedDict` only
   when a library wants plain dicts.
4. **A fixed set of values is a `Literal`.** `Literal["open", "done"]`, not
   loose strings and not a pile of flags.
5. **Raise a named error.** Your own exception class, in its own file. Never
   `raise Exception("...")`.
6. **Absolute imports.** `from app.tasks.task import Task`. Never relative
   imports, never `import *`.
7. **No mutable default arguments.** `items: list[str] | None = None`, never
   `items: list[str] = []`.
8. **`with` for anything that must be closed.** Files, connections, locks.
9. **Private helpers start with `_`.** They sit below the main function and
   nothing outside the file imports them.

---

## Tools

| Job | Tool | Command |
|---|---|---|
| Packages and Python version | uv | `uv add`, `uv run` |
| Formatting | ruff | `ruff format` |
| Lint | ruff | `ruff check` |
| Types | pyright | `pyright` |
| Tests | pytest | `uv run pytest` |

Run all of them before every commit. Settings live in `pyproject.toml`, not
in separate files.

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Function | `create_task` | verb then noun |
| Yes or no function | `can_edit_task` | starts with `is`, `has` or `can` |
| Class | `Task` | noun, `PascalCase` |
| Exception | `TaskNotFound` | says what went wrong |
| Literal type | `TaskStatus` | area then `Status` |
| Literal values | `"open"`, `"done"` | lower case words |
| Constant | `MAX_TITLE_LENGTH` | `UPPER_SNAKE`, at the top of the file |
| Private helper | `_make_slug` | starts with `_` |
| File | `create_task.py` | the thing's name in `snake_case` |
| Test file | `test_create_task.py` | `test_` then the file it tests |
| Test function | `test_create_task_rejects_empty_title` | `test_` then what happens |

`create_task.py` holds `create_task`. `task.py` holds `Task`.
`task_not_found.py` holds `TaskNotFound`.

---

## Folder layout

One folder per area of the work. One thing per file inside it.

    project/
      pyproject.toml
      app/
        __init__.py
        main.py
        settings.py
        tasks/
          __init__.py
          task.py
          task_status.py
          task_not_found.py
          create_task.py
          close_task.py
      tests/
        tasks/
          test_create_task.py
          test_close_task.py

`__init__.py` files stay empty. Never import things into them to make a
shorter path.

---

## A fixed set of values

`app/tasks/task_status.py`

```python
from typing import Literal

type TaskStatus = Literal["open", "done"]
```

---

## Data inside the program

`app/tasks/task.py`

```python
from dataclasses import dataclass
from datetime import datetime

from app.tasks.task_status import TaskStatus


@dataclass(frozen=True)
class Task:
    id: int
    title: str
    status: TaskStatus
    created_at: datetime
```

`frozen=True` means nothing changes a task after it is made. Make a new one
with `dataclasses.replace` instead.

---

## Errors

`app/tasks/task_not_found.py`

```python
class TaskNotFound(Exception):
    def __init__(self, task_id: int) -> None:
        super().__init__(f"Task {task_id} not found")
        self.task_id = task_id
```

---

## A function

`app/tasks/close_task.py`

```python
from dataclasses import replace

from app.tasks.task import Task


def close_task(task: Task) -> Task:
    if task.status == "done":
        return task
    return replace(task, status="done")
```

Bad cases first, main path last, nothing nested.

---

## Tests

`tests/tasks/test_close_task.py`

```python
from datetime import datetime

from app.tasks.close_task import close_task
from app.tasks.task import Task


def test_close_task_marks_task_done() -> None:
    task = Task(id=1, title="Write plan", status="open", created_at=datetime.now())
    assert close_task(task).status == "done"
```

One test checks one thing. The name says what should happen.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    def get_tasks() -> list[Task]:        def get_tasks():
    owner: User | None                    owner: Optional[User]
    raise TaskNotFound(task_id)           raise Exception("not found")
    from app.tasks.task import Task       from .task import *
    tags: list[str] | None = None         tags: list[str] = []
    with open(path) as file:              file = open(path)
    f"Task {task_id}"                     "Task %s" % task_id
    Path("data") / "tasks.json"           os.path.join("data", "tasks.json")
    for index, task in enumerate(tasks):  for i in range(len(tasks)):

---

## Before you commit

- Does every function have types on its arguments and its return?
- Did you use `list`, `dict` and `|` instead of `List`, `Dict` and
  `Optional`?
- Is outside data checked by a Pydantic model before the program uses it?
- Is every error a named exception class in its own file?
- Is every import absolute, with no `import *`?
- Is there a mutable default argument?
- Do `ruff format`, `ruff check`, `pyright` and `pytest` all pass?
