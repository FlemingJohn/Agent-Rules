# Writing FastAPI

Builds on `core/` and `languages/python.md`. This file covers only what is
special to FastAPI.

Written for FastAPI 0.139 with Pydantic 2 and plain SQL. The examples use
PostgreSQL through psycopg 3. For MySQL, see `databases/mysql.md`.

The examples use one made-up API throughout: tasks that can be created and
read.

---

## The layers

A request passes through the same layers every time, in this order:

    request → middleware → dependencies → route → service → query → database

| Layer | Its one job | Never |
|---|---|---|
| `middleware/` | wraps every request: logging, turning errors into replies | business rules |
| `dependencies/` | gives a route what it needs: a database connection, the signed-in user | business rules |
| `routes/` | read the request, call one service, return the reply | SQL, business rules |
| `services/` | the business rules: who may do what, what must be true | `Request`, `HTTPException`, status codes |
| `queries/` | run one SQL statement and return checked data | decisions |
| `schemas/` | the shape of data coming in and going out | logic |
| `errors/` | named errors that services raise | logic |

Each layer calls only the layer below it. A route never calls a query. A
query never calls a service.

---

## The rules

1. **Thin routes.** A route reads the input, calls one service and returns.
   About fifteen lines at most.
2. **Services hold the rules.** They take a connection and plain values,
   return schemas, and raise a named error when something is wrong.
3. **Queries hold one SQL statement each.** Plain SQL with parameters. Never
   an f-string with values in it.
4. **A Pydantic schema for every body.** What comes in and what goes out each
   have one. The return type of the route is the response model.
5. **Dependencies through `Annotated`.** Write each dependency once as a
   named type in `dependencies/`, then use that name in every route.
6. **No `HTTPException` in services or queries.** Services raise named errors.
   One handler turns them into status codes.
7. **`def`, not `async def`.** With a plain sync database driver, routes are
   plain `def`. FastAPI runs them in a thread for you. Use `async def` only
   when every slow call inside is awaited.
8. **Settings come from the environment.** One `Settings` class in
   `config/settings.py`. Never `os.environ` anywhere else.
9. **Inside each layer, one folder per area.**
   `services/tasks/create_task.py`, not one large `task_service.py`.
10. **Start and stop in `lifespan`.** Never `@app.on_event`.

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Route file | `task_routes.py` holds `task_routes` | area then `routes` |
| Route function | `get_task_route`, `create_task_route` | the service it calls, then `route` |
| Service | `get_task`, `create_task` | verb then noun |
| Query | `select_task_by_id`, `insert_task` | starts with the SQL verb |
| Query text | `SELECT_TASK_BY_ID` | the function name in `UPPER_SNAKE` |
| Schema | `Task`, `NewTask` | what the data is |
| Error | `NotFoundError`, `NotAllowedError` | what went wrong, then `Error` |
| Dependency function | `get_connection` | `get` then what it gives |
| Dependency type | `DatabaseConnection` | what the route receives |

Route functions end in `_route`, and services and queries never share a
verb, so nothing clashes when one imports another.

Paths are plural nouns in lower case: `/tasks`, `/tasks/{task_id}`.

---

## Folder layout

    project/
      pyproject.toml
      .env
      app/
        main.py
        lifespan.py
        config/
          settings.py
        database/
          pool.py
          get_connection.py
        dependencies/
          database_connection.py
        middleware/
          log_request.py
          handle_app_error.py
        routes/
          task_routes.py
        services/
          tasks/
            get_task.py
            create_task.py
        queries/
          tasks/
            select_task_by_id.py
            insert_task.py
        schemas/
          tasks/
            task.py
            new_task.py
            task_status.py
        errors/
          app_error.py
          not_found_error.py
          not_allowed_error.py
      migrations/
        001_create_tasks.sql
      tests/
        services/
          tasks/
            test_get_task.py

Run it with `fastapi dev app/main.py`.

---

## The app file

`app/main.py`

```python
from fastapi import FastAPI

from app.errors.app_error import AppError
from app.lifespan import lifespan
from app.middleware.handle_app_error import handle_app_error
from app.middleware.log_request import log_request
from app.routes.task_routes import task_routes

app = FastAPI(lifespan=lifespan)
app.middleware("http")(log_request)
app.add_exception_handler(AppError, handle_app_error)
app.include_router(task_routes)
```

A new area adds one import and one `include_router` line.

---

## Settings

`app/config/settings.py`

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

from app.config.settings import settings

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

`app/dependencies/database_connection.py`

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

## Schemas

`app/schemas/tasks/task.py`

```python
from datetime import datetime

from pydantic import BaseModel

from app.schemas.tasks.task_status import TaskStatus


class Task(BaseModel):
    id: int
    title: str
    status: TaskStatus
    created_at: datetime
```

`app/schemas/tasks/new_task.py`

```python
from pydantic import BaseModel, Field


class NewTask(BaseModel):
    title: str = Field(min_length=1, max_length=200)
```

Checks on input live on the schema as `Field` rules, not as `if` statements
in the route. Bad input gets a 422 before your code runs.

---

## Route

`app/routes/task_routes.py`

```python
from fastapi import APIRouter, status

from app.dependencies.database_connection import DatabaseConnection
from app.schemas.tasks.new_task import NewTask
from app.schemas.tasks.task import Task
from app.services.tasks.create_task import create_task
from app.services.tasks.get_task import get_task

task_routes = APIRouter(prefix="/tasks", tags=["tasks"])


@task_routes.get("/{task_id}")
def get_task_route(task_id: int, connection: DatabaseConnection) -> Task:
    return get_task(connection, task_id)


@task_routes.post("", status_code=status.HTTP_201_CREATED)
def create_task_route(new_task: NewTask, connection: DatabaseConnection) -> Task:
    return create_task(connection, new_task)
```

---

## Service

`app/services/tasks/get_task.py`

```python
from psycopg import Connection
from psycopg.rows import DictRow

from app.errors.not_found_error import NotFoundError
from app.queries.tasks.select_task_by_id import select_task_by_id
from app.schemas.tasks.task import Task


def get_task(connection: Connection[DictRow], task_id: int) -> Task:
    task = select_task_by_id(connection, task_id)
    if task is None:
        raise NotFoundError(f"Task {task_id} not found")
    return task
```

`app/services/tasks/create_task.py`

```python
from psycopg import Connection
from psycopg.rows import DictRow

from app.queries.tasks.insert_task import insert_task
from app.schemas.tasks.new_task import NewTask
from app.schemas.tasks.task import Task


def create_task(connection: Connection[DictRow], new_task: NewTask) -> Task:
    return insert_task(connection, new_task)
```

`create_task` only calls one query today. It still exists, so every route
takes the same path, and the next rule for creating a task has a place to go.

---

## Query

`app/queries/tasks/select_task_by_id.py`

```python
from psycopg import Connection
from psycopg.rows import DictRow

from app.schemas.tasks.task import Task

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

`app/queries/tasks/insert_task.py`

```python
from psycopg import Connection
from psycopg.rows import DictRow

from app.schemas.tasks.new_task import NewTask
from app.schemas.tasks.task import Task

INSERT_TASK = """
    INSERT INTO tasks (title, status)
    VALUES (%(title)s, 'open')
    RETURNING id, title, status, created_at
"""


def insert_task(connection: Connection[DictRow], new_task: NewTask) -> Task:
    row = connection.execute(INSERT_TASK, {"title": new_task.title}).fetchone()
    return Task.model_validate(row)
```

A database row goes straight into the Pydantic schema. No second dataclass
copy of the same shape.

---

## Errors

`app/errors/app_error.py`

```python
class AppError(Exception):
    pass
```

`app/errors/not_found_error.py`

```python
from app.errors.app_error import AppError


class NotFoundError(AppError):
    pass
```

`app/middleware/handle_app_error.py`

```python
from fastapi import Request
from fastapi.responses import JSONResponse

from app.errors.not_allowed_error import NotAllowedError
from app.errors.not_found_error import NotFoundError

STATUS_BY_ERROR = {
    NotAllowedError: 403,
    NotFoundError: 404,
}


def handle_app_error(request: Request, error: Exception) -> JSONResponse:
    status_code = STATUS_BY_ERROR[type(error)]
    return JSONResponse({"detail": str(error)}, status_code=status_code)
```

Every named error is an `AppError`. A new kind of error is one new class and
one new line in the map.

---

## Middleware

`app/middleware/log_request.py`

```python
import logging
import time
from collections.abc import Awaitable, Callable

from fastapi import Request, Response

logger = logging.getLogger(__name__)


async def log_request(
    request: Request, call_next: Callable[[Request], Awaitable[Response]]
) -> Response:
    started_at = time.perf_counter()
    response = await call_next(request)
    milliseconds = round((time.perf_counter() - started_at) * 1000)
    logger.info(f"{request.method} {request.url.path} {response.status_code} {milliseconds}ms")
    return response
```

Middleware is for things every request needs. Anything one route needs, such
as the signed-in user, is a dependency instead.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    route → service → query               route → query
    raise NotFoundError(...) in a         raise HTTPException(404) in a
    service                               service
    def get_task_route(...) -> Task:      @router.get(..., response_model=Task)
    connection: DatabaseConnection        connection = Depends(get_connection)
    WHERE id = %(task_id)s                f"WHERE id = {task_id}"
    title: str = Field(min_length=1)      if not body.title: raise ...
    services/tasks/create_task.py         services/task_service.py with 20
                                          functions
    settings.database_url                 os.environ["DATABASE_URL"]
    lifespan=lifespan                     @app.on_event("startup")
    def create_task_route(...)            async def with a sync driver inside

---

## Before you commit

- Does every request go route → service → query, with no layer skipped or
  added?
- Is every route about fifteen lines or less?
- Does every body in and out have a Pydantic schema, with the return type as
  the response model?
- Is there an `HTTPException` or status code in a service or query?
- Does every new error extend `AppError` and have a line in
  `STATUS_BY_ERROR`?
- Is every query parameterised, with no f-string SQL?
- Is there an `async def` route that calls something slow without `await`?
