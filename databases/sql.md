# Writing SQL

Builds on `core/code-shape.md`, `core/naming.md` and `core/simple-logic.md`.
This file covers SQL that works the same in PostgreSQL and MySQL. Where they
differ, see `databases/postgresql.md` and `databases/mysql.md`.

Plain SQL, no ORM. The app sends SQL text with parameters and reads rows
back.

---

## The rules

1. **Name every column.** Never `SELECT *`.
2. **Always use parameters.** Never build SQL by joining strings or with
   template strings. That is how SQL injection happens.
3. **One statement per query.** Each query function or `.sql` file holds one
   statement.
4. **Keywords in capitals, names in lower case.** `SELECT title FROM tasks`.
5. **One clause per line.** `SELECT`, `FROM`, `JOIN`, `WHERE`, `GROUP BY`,
   `ORDER BY` each start a new line.
6. **Named steps, not nested queries.** A query inside a query inside a query
   becomes a `WITH` block with named steps.
7. **The database guards the data.** `NOT NULL` by default, foreign keys,
   `UNIQUE` and `CHECK` rules in the table, not only in app code.
8. **Changes go in migrations.** Numbered files, run in order, never edited
   after they have run.
9. **More than one change together goes in a transaction.** All of it
   happens, or none of it.

---

## Naming

| Thing | Example | Rule |
|---|---|---|
| Table | `tasks`, `task_comments` | plural noun, `snake_case` |
| Column | `title`, `due_date` | singular noun, `snake_case` |
| Primary key | `id` | always `id` |
| Foreign key | `owner_id`, `task_id` | the role or table in singular, then `_id` |
| Yes or no column | `is_archived`, `has_reminder` | starts with `is`, `has` or `can` |
| Time column | `created_at`, `closed_at` | past tense verb, then `_at` |
| Date column | `due_date` | ends with `_date` |
| Status column | `status` | one column, a fixed set of values |
| Index | `tasks_owner_id_index` | table, columns, then `index` |
| Unique rule | `users_email_unique` | table, columns, then `unique` |
| Migration file | `003_add_due_date_to_tasks.sql` | number, then what it does |

No table aliases like `t` or `u`. Write the table name. When the same table
is joined twice, name each alias for its role: `owners`, `assignees`.

---

## Folder layout

    migrations/
      001_create_users.sql
      002_create_tasks.sql
      003_add_due_date_to_tasks.sql

Queries the app runs live next to the code that uses them, one per file. How
they sit in the code depends on the framework file.

---

## A table

`migrations/002_create_tasks.sql`, written for PostgreSQL:

```sql
CREATE TABLE tasks (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    owner_id BIGINT NOT NULL REFERENCES users (id),
    title VARCHAR(200) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'open',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT tasks_status_check CHECK (status IN ('open', 'done'))
);

CREATE INDEX tasks_owner_id_index ON tasks (owner_id);
```

A status is a text column with a `CHECK` rule. It is easy to read in a query
result and easy to add a value to later. The id and time types differ in
MySQL. See `databases/mysql.md`.

Every foreign key gets an index. So does every column you often filter by.

---

## A query

```sql
SELECT
    tasks.id,
    tasks.title,
    tasks.status,
    users.name AS owner_name
FROM tasks
JOIN users ON users.id = tasks.owner_id
WHERE tasks.status = 'open'
ORDER BY tasks.created_at DESC
LIMIT 50
```

Join with `JOIN ... ON`. Never list tables with commas and join in
`WHERE`.

---

## Named steps

    good
    ──────────────────────────────────────────────────────────────
    WITH open_task_counts AS (
        SELECT owner_id, COUNT(*) AS open_task_count
        FROM tasks
        WHERE status = 'open'
        GROUP BY owner_id
    )
    SELECT users.name, open_task_counts.open_task_count
    FROM users
    JOIN open_task_counts ON open_task_counts.owner_id = users.id
    WHERE open_task_counts.open_task_count > 10

    bad
    ──────────────────────────────────────────────────────────────
    SELECT name, (SELECT COUNT(*) FROM tasks WHERE owner_id = u.id AND
    status = 'open') AS c FROM users u WHERE (SELECT COUNT(*) FROM tasks
    WHERE owner_id = u.id AND status = 'open') > 10

---

## Migrations

- One change per migration file: create one table, add one column, add one
  index.
- Never edit a migration that has run anywhere. Write a new one that fixes
  it.
- A new `NOT NULL` column on a table with rows needs a `DEFAULT`, or it
  fails.
- Never delete a column the running code still reads. Stop reading it first,
  then remove it in a later migration.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    SELECT id, title FROM tasks           SELECT * FROM tasks
    WHERE id = $1                         "WHERE id = " + taskId
    JOIN users ON users.id = ...          FROM tasks, users WHERE ...
    FROM tasks                            FROM tasks t
    status VARCHAR(20) + CHECK            is_open, is_done, is_late columns
    owner_id BIGINT NOT NULL              owner_id BIGINT
    WITH named_step AS (...)              three nested subqueries
    new migration file                    editing 002_create_tasks.sql
    created_at                            creation_timestamp, dt_created

---

## Before you commit

- Is there a `SELECT *`?
- Is every value passed as a parameter, never joined into the text?
- Does every column that must have a value say `NOT NULL`?
- Does every foreign key have an index?
- Is there a query nested more than one level deep? Use `WITH`.
- Did you edit a migration that has already run?
- Are several changes that belong together inside one transaction?
