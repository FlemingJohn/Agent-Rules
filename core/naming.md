# Naming

Plain words that say what the thing does. No cleverness, no poetry, no
shorthand.

---

## The rules

1. **Verbs for things that do.** `get`, `set`, `make`, `start`, `end`,
   `update`, `handle`, `check`, `send`, `draw`, `pick`, `create`, `delete`.
2. **Nouns for things that are.** `task`, `total`, `status`, `results`.
3. **`is`, `has` or `can` for yes or no.** `is_ready`, `has_loaded`,
   `can_edit`.
4. **No abbreviations.** `configuration` not `config`, `message` not `msg`,
   `button` not `btn`, `value` not `val`, `response` not `resp`.
5. **Names the outside world already uses are fine.** `id`, `url`, `api`,
   `html`, `json`, `sql`, and names a framework fixes for you, such as `req`
   and `res` in Express, `props` in React or `page.tsx` in Next.js.
6. **Name things for what they hold or do,** never for what they look like or
   where they sit.

---

## Examples

    good                          bad
    ─────────────────────────────────────────────────────
    get_task_by_id                fetch_the_thing
    get_total                     what_it_all_comes_to
    start_request                 begin_the_journey
    can_edit                      permission_allows
    task_count                    how_many_things
    is_ready                      all_systems_go
    has_finished                  the_end_has_come
    tasks                         data
    task                          item, obj, x
    status                        st, flag
    task_routes.py                routes2.py
    TaskCard                      BlueBox

---

## Names that say nothing

Never use these as a whole name: `data`, `info`, `item`, `obj`, `thing`,
`stuff`, `temp`, `result` alone, `helper`, `utils`, `manager`, `handler`
alone, `process`, `do_work`, `node1`, `step_a`.

Add the noun that says what it is: `task_data` is still vague, `new_task` is
not.

---

## Same thing, same name

One idea gets one word across the whole project. If it is a `task` in the
database, it is a `task` in the code, the API and the screen. Never `task`
in one file, `job` in another and `todo` in a third.

---

## Case

Each language has its own case. Follow the language file:

| Language | Variables and functions | Types and classes | Constants | Files |
|---|---|---|---|---|
| Python | `snake_case` | `PascalCase` | `UPPER_SNAKE` | `snake_case.py` |
| TypeScript | `camelCase` | `PascalCase` | `UPPER_SNAKE` | same case as the thing |
| JavaScript | `camelCase` | `PascalCase` | `UPPER_SNAKE` | same case as the thing |
| SQL | `snake_case` | — | — | `snake_case.sql` |

---

## Before you commit

- Does every name say what the thing does or holds?
- Is there an abbreviation that is not a name the outside world uses?
- Is there a `data`, `item`, `utils` or `helper` on its own?
- Does one idea have two names anywhere?
