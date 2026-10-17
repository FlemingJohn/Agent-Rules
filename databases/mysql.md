# Writing for MySQL

Builds on `databases/sql.md`. This file covers only what is special to
MySQL.

Written for MySQL 8.4 LTS or newer.

---

## The rules

1. **Ids are `BIGINT UNSIGNED AUTO_INCREMENT`.** Foreign keys use the same
   type, `BIGINT UNSIGNED`.
2. **Times are `DATETIME`, stored in UTC.** Never `TIMESTAMP`, which stops
   working in 2038. The app always writes and reads UTC.
3. **Every table is InnoDB with `utf8mb4`.** These are the defaults in MySQL
   8. Never change them.
4. **Yes or no is `BOOLEAN`.** MySQL stores it as `TINYINT(1)`, which
   comes back as `1` or `0`.
5. **No `RETURNING`.** After an insert, read the new id from the driver,
   then select the row.
6. **Insert or update with `ON DUPLICATE KEY UPDATE`.** Never check first and
   then insert.
7. **Strict mode stays on.** Never turn off `STRICT_TRANS_TABLES`. Without
   it, MySQL cuts off long text and stores bad dates without an error.
8. **No reserved words as names.** If a name needs backticks, rename it.

---

## Types

| Value | Type |
|---|---|
| Id | `BIGINT UNSIGNED AUTO_INCREMENT` |
| Foreign key | `BIGINT UNSIGNED` |
| Short text with a limit | `VARCHAR(200)` |
| Long text | `TEXT` |
| Yes or no | `BOOLEAN` |
| Moment in time | `DATETIME`, in UTC |
| Calendar date | `DATE` |
| Money | `DECIMAL(12, 2)`, or whole cents as `BIGINT` |
| Status | `VARCHAR(20)` with a `CHECK` rule |

Never `FLOAT` or `DOUBLE` for money. Never MySQL's `ENUM` type. Adding a
value to it changes the table.

---

## A table

`migrations/002_create_tasks.sql`

```sql
CREATE TABLE tasks (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    owner_id BIGINT UNSIGNED NOT NULL,
    title VARCHAR(200) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'open',
    created_at DATETIME NOT NULL DEFAULT (UTC_TIMESTAMP()),
    CONSTRAINT tasks_owner_id_foreign FOREIGN KEY (owner_id) REFERENCES users (id),
    CONSTRAINT tasks_status_check CHECK (status IN ('open', 'done'))
);

CREATE INDEX tasks_owner_id_index ON tasks (owner_id);
```

---

## Getting the new row back

Insert, take the new id from the driver, then run the normal select for one
row:

```sql
INSERT INTO tasks (owner_id, title)
VALUES (?, ?)
```

| Driver | Language | New id |
|---|---|---|
| PyMySQL | Python | `cursor.lastrowid` |
| mysql2 | Node | `result.insertId` |

---

## Insert or update

```sql
INSERT INTO task_settings (task_id, has_reminder)
VALUES (?, ?) AS new_settings
ON DUPLICATE KEY UPDATE has_reminder = new_settings.has_reminder
```

Name the new row with `AS`. Never use the old `VALUES(column)` form.

---

## Parameters by driver

| Driver | Language | Parameter |
|---|---|---|
| PyMySQL | Python | `%(task_id)s` with a dict |
| mysql2 | Node | `?` with an array |

---

## With FastAPI

Use PyMySQL with `DictCursor`, so rows come back as dicts. Open one
connection per request.

`app/database/get_connection.py`

```python
from collections.abc import Iterator

import pymysql
from pymysql.connections import Connection
from pymysql.cursors import DictCursor

from app.config.settings import settings


def get_connection() -> Iterator[Connection]:
    connection = pymysql.connect(
        host=settings.database_host,
        user=settings.database_user,
        password=settings.database_password,
        database=settings.database_name,
        cursorclass=DictCursor,
    )
    with connection:
        yield connection
        connection.commit()
```

A query function then runs its SQL through a cursor:

```python
def select_task_by_id(connection: Connection, task_id: int) -> Task | None:
    with connection.cursor() as cursor:
        cursor.execute(SELECT_TASK_BY_ID, {"task_id": task_id})
        row = cursor.fetchone()
    if row is None:
        return None
    return Task.model_validate(row)
```

---

## With Node

Use `mysql2/promise` and one pool for the whole app.

```ts
import mysql from "mysql2/promise";

export const pool = mysql.createPool({
  uri: settings.databaseUrl,
  timezone: "Z",
});
```

`timezone: "Z"` keeps every `DATETIME` in UTC.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    BIGINT UNSIGNED AUTO_INCREMENT        INT AUTO_INCREMENT
    DATETIME in UTC                       TIMESTAMP
    VARCHAR(20) + CHECK                   ENUM('open', 'done')
    lastrowid / insertId                  SELECT MAX(id) FROM tasks
    ON DUPLICATE KEY UPDATE               SELECT, then INSERT or UPDATE
    VALUES (...) AS new_settings          VALUES(has_reminder)
    DECIMAL(12, 2)                        DOUBLE for money
    order_date                            `order` as a column name

---

## Before you commit

- Is every id `BIGINT UNSIGNED AUTO_INCREMENT`, with matching foreign keys?
- Is every time a `DATETIME` written in UTC?
- Is there an `ENUM` column? Make it `VARCHAR` with a `CHECK`.
- Does any insert read the new id with `MAX(id)`?
- Does any name need backticks? Rename it.
