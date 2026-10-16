# Code shape

These rules apply to every language and framework in this repo. The language
and framework files build on them and never repeat them.

---

## The rules

1. **Small files, one job each.** If a file does two things, it is two files.
2. **One thing per file.** Every type, class, component, hook, schema, query
   and router sits alone in its own file, named after it.
3. **Short functions.** A function past about thirty lines is doing too much.
   Split it. A file past about two hundred lines has the same problem.
4. **No comments. None.** Not above a function, not at the end of a line, not
   to mark a section. No docstrings in Python, no JSDoc in TypeScript or
   JavaScript. If a line needs explaining, the names are wrong. Fix the names.
5. **Main path first.** A reader opens a file and follows it top to bottom.
   The main function comes first. Its private helpers sit below it.
6. **Reuse before writing.** Look for an existing function, type, component or
   query before making a new one. Two things that do the same job should be
   one thing. Two buttons that look the same should be one button.

---

## One thing per file

The file holds one thing that other files use. That thing may have private
helpers that only it uses. They sit below it in the same file, and nothing
else imports them.

    good                                  bad
    ──────────────────────────────────────────────────────────────
    task.py           holds Task          models.py   holds 9 classes
    TaskCard.tsx      holds TaskCard      types.ts    holds every type
    useTasks.ts       holds useTasks      utils.ts    holds anything
    select_task.sql   one query           queries.sql holds 40 queries

Never a shared `types`, `utils`, `helpers`, `common` or `misc` file. A pile
file grows forever and nobody can tell what is in it from its name.

---

## Comments

    bad                                          good
    ──────────────────────────────────────────────────────────────────
    # check if user can edit                     if can_edit(user, task):
    if user.role == "admin" or task.owner_id     ...
       == user.id:

    // total in cents                            const totalInCents = ...
    const t = ...

The only lines allowed are the ones a language or tool demands, such as a
shebang, `"use client"`, `"use server"` or a license header a tool requires.

---

## Main path first

    good                                  bad
    ──────────────────────────────────────────────────────────────
    def create_task(...):                 def _clean_title(...):
        title = _clean_title(...)             ...
        ...                               def _check_owner(...):
                                              ...
    def _clean_title(...):                def create_task(...):
        ...                                   ...

The reader sees what the file does before seeing how.

---

## What the code should feel like

Open a folder and you should see the steps of that part of the work, in
order, each one readable on its own.

Open a screen and you should see what it shows and what happens when you
press things. Nothing else.

If you cannot tell what a file does from its name and its first ten lines,
it needs breaking up.

---

## Before you commit

- Does each file hold one thing, named after it?
- Is any function over about thirty lines, or any file over about two
  hundred?
- Is there a single comment, docstring or JSDoc block? Remove it and fix the
  names.
- Does each file read top to bottom with the main function first?
- Did you write something that already existed?
