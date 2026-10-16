# Writing FastAPI

Builds on `core/` and `languages/python.md`. This file covers only what is
special to FastAPI.

Written for FastAPI 0.139 with Pydantic 2 and plain SQL. The examples use
PostgreSQL through psycopg 3. For MySQL, see `databases/mysql.md`.

The examples use one made-up API throughout: tasks that can be created and
read.

---

## The rules

1. **Thin endpoints.** An endpoint reads the input, calls one or two
   functions and returns. About fifteen lines at most.
2. **A Pydantic model for every body.** What comes in and what goes out each
   have a model. The return type of the endpoint is the response model.
3. **One router per area.** `task_router.py` holds every `/tasks` endpoint.
   The router is the one thing in that file.
4. **Dependencies through `Annotated`.** Write the dependency once as a named
   type, then use that name in every endpoint.
5. **`def`, not `async def`.** With a plain sync database driver, endpoints
   are plain `def`. FastAPI runs them in a thread for you. Use `async def`
   only when every slow call inside is awaited.
6. **Settings come from the environment.** One `Settings` class from
   `pydantic-settings`. Never `os.environ` scattered through the code.
7. **SQL is plain and has parameters.** Never build a query with an f-string.
8. **HTTP errors only in the router.** Query functions return `None` or raise
   their own error. Only the endpoint turns that into an `HTTPException`.
9. **Start and stop in `lifespan`.** Never `@app.on_event`.

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Router | `task_router` | area then `router` |
| Read one | `get_task` | `get` then the noun |
| Read many | `list_tasks` | `list` then the plural noun |
| Create | `create_task` | `create` then the noun |
| Change | `edit_task` | `edit` then the noun |
| Remove | `remove_task` | `remove` then the noun |
| Query function | `select_task_by_id`, `insert_task` | starts with the SQL verb |
| Query text | `SELECT_TASK_BY_ID` | the function name in `UPPER_SNAKE` |
| Response model | `Task` | the noun |
| Request model | `NewTask`, `TaskChange` | what the caller sends |
| Dependency | `get_connection` | `get` then what it gives |
| Dependency type | `DatabaseConnection` | what the endpoint receives |

Endpoints and query functions never share a verb, so the router can import
both without a name clash.

Paths are plural nouns in lower case: `/tasks`, `/tasks/{task_id}`. Never
verbs such as `/getTask`.

---

## Folder layout

    project/
      pyproject.toml
      .env
      app/
        main.py
        settings.py
        lifespan.py
        database/
          pool.py
          get_connection.py
          database_connection.py
        tasks/
          task.py
          new_task.py
          task_status.py
          select_task_by_id.py
          insert_task.py
          task_router.py
      migrations/
        001_create_tasks.sql
      tests/
        tasks/
          test_task_router.py

Run it with `fastapi dev app/main.py`.

---

## The app file

`app/main.py`

```python
from fastapi import FastAPI

from app.lifespan import lifespan
from app.tasks.task_router import task_router

app = FastAPI(lifespan=lifespan)
app.include_router(task_router)
```

Nothing else goes in it. A new area adds one import and one
`include_router` line.

---

## Settings

`app/settings.py`

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    database_url: str


settings = Settings()
```

The app fails at start-up if a setting is missing, not halfway through a
request.

---

## The database connection

`app/database/pool.py`

```python
from psycopg import Connection
from psycopg.rows import DictRow, dict_row
from psycopg_pool import ConnectionPool

from app.settings import settings

pool = ConnectionPool(
    settings.database_url,
    connection_class=Connection[DictRow],
    kwargs={"row_factory": dict_row},
    open=False,
)
```

`app/lifespan.py`

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.database.pool import pool


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    pool.open()
    yield
    pool.close()
```

`app/database/get_connection.py`

```python
from collections.abc import Iterator

from psycopg import Connection
from psycopg.rows import DictRow

from app.database.pool import pool


def get_connection() -> Iterator[Connection[DictRow]]:
    with pool.connection() as connection:
        yield connection
```

The connection commits when the request ends, and rolls back if it fails.

`app/database/database_connection.py`

```python
from typing import Annotated

from fastapi import Depends
from psycopg import Connection
from psycopg.rows import DictRow

from app.database.get_connection import get_connection

DatabaseConnection = Annotated[Connection[DictRow], Depends(get_connection)]
```

Dependency types use plain assignment, not the `type` statement, so FastAPI
can read them.

---

## Models

`app/tasks/task.py`

```python
from datetime import datetime

from pydantic import BaseModel

from app.tasks.task_status import TaskStatus


class Task(BaseModel):
    id: int
    title: str
    status: TaskStatus
    created_at: datetime
```

`app/tasks/new_task.py`

```python
from pydantic import BaseModel, Field


class NewTask(BaseModel):
    title: str = Field(min_length=1, max_length=200)
```

Checks on input live on the model as `Field` rules, not as `if` statements
in the endpoint. Bad input gets a 422 before your code runs.

In a FastAPI app, a database row goes straight into the Pydantic model the
API returns. No second dataclass copy of the same shape.

---

## Query functions

One query per file. The SQL sits at the top as a constant, and the function
below it runs it.

`app/tasks/select_task_by_id.py`

```python
from psycopg import Connection
from psycopg.rows import DictRow

from app.tasks.task import Task

SELECT_TASK_BY_ID = """
    SELECT id, title, status, created_at
    FROM tasks
    WHERE id = %(task_id)s
"""


def select_task_by_id(connection: Connection[DictRow], task_id: int) -> Task | None:
    row = connection.execute(SELECT_TASK_BY_ID, {"task_id": task_id}).fetchone()
    if row is None:
        return None
    return Task.model_validate(row)
```

`app/tasks/insert_task.py`

```python
from psycopg import Connection
from psycopg.rows import DictRow

from app.tasks.new_task import NewTask
from app.tasks.task import Task

INSERT_TASK = """
    INSERT INTO tasks (title, status)
    VALUES (%(title)s, 'open')
    RETURNING id, title, status, created_at
"""


def insert_task(connection: Connection[DictRow], new_task: NewTask) -> Task:
    row = connection.execute(INSERT_TASK, {"title": new_task.title}).fetchone()
    return Task.model_validate(row)
```

Query functions know nothing about HTTP. They take a connection and plain
values, and return models.

---

## The router

`app/tasks/task_router.py`

```python
from fastapi import APIRouter, HTTPException, status

from app.database.database_connection import DatabaseConnection
from app.tasks.insert_task import insert_task
from app.tasks.new_task import NewTask
from app.tasks.select_task_by_id import select_task_by_id
from app.tasks.task import Task

task_router = APIRouter(prefix="/tasks", tags=["tasks"])


@task_router.get("/{task_id}")
def get_task(task_id: int, connection: DatabaseConnection) -> Task:
    task = select_task_by_id(connection, task_id)
    if task is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, f"Task {task_id} not found")
    return task


@task_router.post("", status_code=status.HTTP_201_CREATED)
def create_task(new_task: NewTask, connection: DatabaseConnection) -> Task:
    return insert_task(connection, new_task)
```

Each endpoint reads like a sentence: get the task, stop if it is missing,
return it.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    def get_task(...) -> Task:            @router.get(..., response_model=Task)
                                          def get_task(...):
    connection: DatabaseConnection        connection = Depends(get_connection)
    WHERE id = %(task_id)s                f"WHERE id = {task_id}"
    title: str = Field(min_length=1)      if not body.title: raise ...
    status.HTTP_404_NOT_FOUND             404
    raise HTTPException in the router     raise HTTPException in a query file
    settings.database_url                 os.environ["DATABASE_URL"]
    lifespan=lifespan                     @app.on_event("startup")
    def create_task(...)                  async def create_task(...) with a
                                          sync driver inside
    /tasks/{task_id}                      /getTask?id=1

---

## Before you commit

- Is every endpoint about fifteen lines or less?
- Does every body in and out have a Pydantic model, with the return type as
  the response model?
- Does every endpoint use a named dependency type instead of a bare
  `Depends`?
- Is every query parameterised, with no f-string SQL?
- Is every `HTTPException` raised in a router file and nowhere else?
- Are settings read only through `Settings`?
- Is there an `async def` endpoint that calls something slow without
  `await`?
