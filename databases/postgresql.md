# Writing for PostgreSQL

Builds on `databases/sql.md`. This file covers only what is special to
PostgreSQL.

Written for PostgreSQL 18.

---

## The rules

1. **Ids are identity columns.** `BIGINT GENERATED ALWAYS AS IDENTITY
   PRIMARY KEY`. Never `SERIAL`.
2. **Times are `TIMESTAMPTZ`.** Never plain `TIMESTAMP`. PostgreSQL stores
   the moment in UTC and converts it for you.
3. **Text is `TEXT` or `VARCHAR(n)`.** Use `VARCHAR(n)` when there is a real
   limit, such as a title. Never `CHAR(n)`.
4. **Yes or no is `BOOLEAN`.**
5. **Get the new row back with `RETURNING`.** No second query to read what
   you just wrote.
6. **Insert or update with `ON CONFLICT`.** Never check first and then insert.
7. **Lower case names, never quoted.** `created_at`, not `"createdAt"`.
   Quoted names must be quoted forever after.
8. **`JSONB` only for data you never filter or join on.** If you query it,
   it should be a column.

---

## Types

| Value | Type |
|---|---|
| Id | `BIGINT GENERATED ALWAYS AS IDENTITY` |
| Foreign key | `BIGINT` |
| Short text with a limit | `VARCHAR(200)` |
| Long text | `TEXT` |
| Yes or no | `BOOLEAN` |
| Moment in time | `TIMESTAMPTZ` |
| Calendar date | `DATE` |
| Money | `NUMERIC(12, 2)`, or whole cents as `BIGINT` |
| Status | `VARCHAR(20)` with a `CHECK` rule |
| Loose extra data | `JSONB` |

Never `FLOAT` or `REAL` for money.

---

## Getting the new row back

```sql
INSERT INTO tasks (owner_id, title)
VALUES ($1, $2)
RETURNING id, title, status, created_at
```

---

## Insert or update

```sql
INSERT INTO task_settings (task_id, has_reminder)
VALUES ($1, $2)
ON CONFLICT (task_id) DO UPDATE
SET has_reminder = EXCLUDED.has_reminder
```

`EXCLUDED` holds the row you tried to insert.

---

## Parameters by driver

| Driver | Language | Parameter |
|---|---|---|
| psycopg 3 | Python | `%(task_id)s` with a dict |
| node-postgres (`pg`) | Node | `$1`, `$2` with an array |

---

## Names the code expects

The database uses `snake_case`. TypeScript uses `camelCase`. Turn one into
the other in the `SELECT`, so the row already has the shape the code wants:

```sql
SELECT id, title, created_at AS "createdAt"
FROM tasks
WHERE id = $1
```

The quotes here only name the result. The column itself stays
`created_at`. Python code keeps `snake_case` and needs no aliases.

---

## Common mistakes

    good                                  bad
    ──────────────────────────────────────────────────────────────
    GENERATED ALWAYS AS IDENTITY          SERIAL
    TIMESTAMPTZ                           TIMESTAMP
    INSERT ... RETURNING id               INSERT, then SELECT max(id)
    ON CONFLICT (task_id) DO UPDATE       SELECT, then INSERT or UPDATE
    created_at                            "createdAt" as a column name
    NUMERIC(12, 2)                        REAL for money
    a real column                         JSONB you filter on every query

---

## Before you commit

- Is every id an identity column and every time a `TIMESTAMPTZ`?
- Does every insert that needs the new row use `RETURNING`?
- Is there a check-then-insert that should be `ON CONFLICT`?
- Is any column name quoted or in `camelCase`?
- Is money stored as `NUMERIC` or whole cents?
