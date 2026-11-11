# Worked example

One full run, from the request through gate closure. Illustration only — every rule this
example follows is defined in `SKILL.md` or `references/checklist-format.md`, and where
the two disagree with anything below, they are right and this file is stale.

The project is a small Python CLI with a `CONVENTIONS.md` that requires type hints on
public functions and pytest coverage for new modules.

## The request

> "Add a command that exports the task list to CSV. I need it to spit out a CSV I can
> open in Excel, and make it fast — the list is big now. Also make sure it looks
> professional."

## Phase A — Contract

**Finding the specification.** No change proposal, no task list, no issue. Requirements
come from the message above, quoted verbatim.

**Reading the conventions.** `CONVENTIONS.md` requires type hints on public functions and
a pytest module for anything new. Those become `implied` rows.

**Triage.**

- R1, R2 — one reading each. `CLEAR`.
- R3 "make it fast" — streaming, indexing, and parallelising are different work at
  different cost. `AMBIGUOUS`.
- R4 "looks professional" — could mean the CSV's formatting, or the terminal output of
  the command. Different work. `AMBIGUOUS`.
- R5, R6 — convention rows, one reading each. `CLEAR`.

**The one question round.**

> Two things need a decision before I start.
>
> Q1 — R3, "make it fast": the list is ~200k rows in the fixtures. Which?
>   (a) Stream rows to the file rather than building it in memory
>   (b) Add an index to the query behind the list
>   → Recommend (a) — memory is what breaks first at that size.
>
> Q2 — R4, "looks professional": the CSV itself, or the command's terminal output?
>   (a) The CSV — proper quoting, header row, stable column order
>   (b) The terminal output — progress, summary line
>   → Recommend (a) — Excel is the stated destination.

User answers: "a and a, yeah".

R3 and R4 are rewritten to what was chosen, with the answer as their Source.

**Sign-off.** The table is presented. Every Status and Evidence cell is empty, because
nothing has been checked yet, and the header says the contract is not yet agreed:

```
Contract: conversation
Sign-off: awaiting
```

| ID | Requirement | Source | Type | Status | Evidence |
|----|-------------|--------|------|--------|----------|
| R1 | An export command exists | "Add a command that exports the task list to CSV" | explicit | | |
| R2 | Output is CSV openable in Excel | "spit out a CSV I can open in Excel" | explicit | | |
| R3 | Rows stream to file rather than buffering in memory | "a and a, yeah" (Q1a) | explicit | | |
| R4 | CSV has quoting, header row, stable column order | "a and a, yeah" (Q2a) | explicit | | |
| R5 | Public functions carry type hints | `CONVENTIONS.md` | implied | | |
| R6 | New module has a pytest module | `CONVENTIONS.md` | implied | | |

Were the session to end here and resume tomorrow, On Invoke would find six open rows and
`Sign-off: awaiting`, and return to the sign-off step rather than starting over or
reporting a verdict.

The implied rows are called out explicitly:

> R5 and R6 come from `CONVENTIONS.md`, not from your message — type hints on public
> functions, and a pytest module for the new code. Say if you want either waived.

User: "keep both".

The header's `Sign-off:` field becomes `signed`, the file is written to
`tasks/spec-check.md`, and the user is told the checklist is a working artifact they can
keep or drop as they prefer.

## Phase B — Work

The command is built. No checklist output during the work.

At one point the user says: "actually put the header row in bold — no wait, CSV can't do
that, forget it." R4 is untouched; nothing was added or changed, so nothing is recorded.

On finishing the export module, the work R1 asks for is complete — that is trigger 4, so the table is
restated and the file saved. Then the tests are written, and the work reaches the point
of reporting done — trigger 2, which runs the gate.

## Phase C — Gate, cycle 1

Read the diff fresh. Forward coverage first, then reverse.

| ID | Requirement | Source | Type | Status | Evidence |
|----|-------------|--------|------|--------|----------|
| R1 | An export command exists | "Add a command that exports the task list to CSV" | explicit | PASS | `cli.py:82` |
| R2 | Output is CSV openable in Excel | "spit out a CSV I can open in Excel" | explicit | PASS | `pytest -k excel → 2 passed` |
| R3 | Rows stream to file rather than buffering in memory | "a and a, yeah" (Q1a) | explicit | PASS | `export.py:31` |
| R4 | CSV has quoting, header row, stable column order | "a and a, yeah" (Q2a) | explicit | FAIL | `export.py:47` — column order from dict iteration, varies per run |
| R5 | Public functions carry type hints | `CONVENTIONS.md` | implied | PASS | `export.py:24,31,47` |
| R6 | New module has a pytest module | `CONVENTIONS.md` | implied | PASS | `tests/test_export.py` |
| R7 | Retry logic added to the sync client | `sync.py:112-140` | unrequested | UNREQUESTED | |

```
VERDICT: 7 rows — 5 PASS, 1 FAIL (R4), 1 UNREQUESTED (R7). Gate open. Cycle 1.
```

Two things to report:

- **R4 fails.** Column order comes from dict iteration, so it varies between runs. The
  requirement asked for stable order.
- **R7 is unrequested.** The diff adds retry logic to the sync client. No requirement
  covers it. It stays in place — reported, not reverted:

> The diff also adds retry logic to `sync.py:112-140`, which nothing in the contract asked
> for. Keep it, or take it out?

User: "oh that was leftover from yesterday, leave it, I'll deal with it separately."

R7 stays `UNREQUESTED` — the user kept the change rather than dropping a requirement, so
`WAIVED` would describe something that did not happen. Its Evidence cell becomes
`USER-CONFIRMED`.

Escalation check: the change touches four files and the contract holds seven rows, both
under the thresholds; no row rests on `USER-CONFIRMED` alone; and cycle 1 produced a
failure, so it was not a clean first pass. No escalation triggered.

## Fix

Column order is pinned to an explicit field list at `export.py:47`. Gate runs again.

## Phase C — Gate, cycle 2

The full table again, not only the row that changed:

| ID | Requirement | Source | Type | Status | Evidence |
|----|-------------|--------|------|--------|----------|
| R1 | An export command exists | "Add a command that exports the task list to CSV" | explicit | PASS | `cli.py:82` |
| R2 | Output is CSV openable in Excel | "spit out a CSV I can open in Excel" | explicit | PASS | `pytest -k excel → 2 passed` |
| R3 | Rows stream to file rather than buffering in memory | "a and a, yeah" (Q1a) | explicit | PASS | `export.py:31` |
| R4 | CSV has quoting, header row, stable column order | "a and a, yeah" (Q2a) | explicit | PASS | `export.py:47` — explicit field list; `pytest -k column_order → 1 passed` |
| R5 | Public functions carry type hints | `CONVENTIONS.md` | implied | PASS | `export.py:24,31,47` |
| R6 | New module has a pytest module | `CONVENTIONS.md` | implied | PASS | `tests/test_export.py` |
| R7 | Retry logic added to the sync client | `sync.py:112-140` | unrequested | UNREQUESTED | `USER-CONFIRMED` |

```
VERDICT: 7 rows — 6 PASS, 1 UNREQUESTED. Gate closed. Cycle 2.
```

Both cycles print every row, which is what makes them comparable. A cycle-2 output showing
only R4 would hide that R7 is still unrequested work sitting in the diff, and would leave
the reader unable to tell cycle 2's result from cycle 1's without holding both in their
head.

Escalation check on cycle 2: still four files and seven rows, no `USER-CONFIRMED`-only
rows, and this was not a first cycle. No escalation.

Work reported done.

## What a clean first pass would have looked like

Had cycle 1 returned all seven rows `PASS` with no failures, the third escalation trigger
in Phase C step 4 would have fired on that alone, and the gate would not have closed on
it. An independent reviewer — given the contract and the diff, and nothing about why the
code was written that way — would have checked the table before the verdict stood. On a
platform with no way to obtain one, the verdict still stands, with a line stating the
review was not independent.
