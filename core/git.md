# Git

---

## The rules

1. **One file per commit.** If a change touches four files, that is four
   commits.
2. **Subject line only.** No body, no description, no footer, no trailing
   lines of any kind.
3. **Never add a co-author line, a session link or a "generated with" note.**
4. **Kind, colon, plain verb.** Lower case, no full stop.

---

## Kinds

| Kind | Use for |
|---|---|
| `feat` | something new the product can do |
| `fix` | something that was wrong and now is not |
| `style` | how it looks or how the code is formatted, with no change in behaviour |
| `refactor` | a change in shape with no change in behaviour |
| `docs` | writing for people, not the program |
| `test` | tests only |
| `chore` | folders, tooling, packages |

---

## Verbs

`add`, `draw`, `fix`, `move`, `split`, `remove`, `rename`, `write`, `stop`,
`correct`, `update`.

---

## Examples

    good                                  bad
    ──────────────────────────────────────────────────────────────
    feat: add the task type               Added Task type.
    feat: draw the status chip            feat: StatusChip component + styles
    fix: correct the rounding             fix: fixed bug
    fix: stop the total going negative    WIP
    style: move colours to tokens         update stuff
    refactor: split the list builder      refactor: misc cleanup
    docs: write the plan                  docs: Update README.md
    chore: add folders                    chore: various changes

---

## Before you commit

- Does the commit hold exactly one file?
- Is the message a single line, starting with a kind and a plain verb?
- Is there any body, co-author line, link or note? Remove it.
