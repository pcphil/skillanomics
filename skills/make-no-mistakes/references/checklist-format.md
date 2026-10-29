# Checklist format

This file defines the header, the columns, the six status values, the three evidence
forms, and the verdict line. It does not define when a row is open or when the gate may
close — `SKILL.md` owns those, under On Invoke and Phase C. Where a rule here would need
to repeat one of them, it points there instead.

## The header

Two fields, above the table:

```
Contract: openspec/changes/add-export/specs/export/spec.md
Sign-off: signed
```

- **Contract** — where the requirements came from: the path of the specification artifact,
  or `conversation` when they were drawn from what the user wrote.
- **Sign-off** — `awaiting` or `signed`. On Invoke reads this field to decide where a
  resumed run continues, so it is the one piece of phase state the file carries. Nothing
  else records it: every Status cell is empty from capture until verification, which is
  equally true of a contract still being assembled and of one signed hours ago.

## The table

| ID | Requirement | Source | Type | Status | Evidence |
|----|-------------|--------|------|--------|----------|
| R1 | Export writes a CSV file | "I need it to spit out a CSV" | explicit | PASS | `export.py:44` |
| R2 | Columns are quoted per RFC 4180 | `CONVENTIONS.md` | implied | FAIL | `export.py:51` — no quoting applied |
| R3 | Output reads as clean, professional | "make it look professional" | explicit | UNVERIFIABLE |  |
| R4 | Column order is stable across runs | "same every time" | explicit |  |  |

R3 and R4 show the two ways a cell can be blank, and they mean different things. R4 has
not been checked yet — empty Status, empty Evidence. R3 has been checked and found to be
something no file or command can settle, so its Status is filled and only its Evidence
waits. Leave both cells genuinely empty in these cases: a dash, `n/a`, or `TBD` reads as
a value, and an empty Status cell is one of the things On Invoke tests for.

## Columns

### ID

A stable label — `R1`, `R2`, `R3` — unique within the file. Assign once and never
renumber, even when a row is superseded. A reader returning to the file after a compaction
needs `R2` to still mean what it meant when it was signed off.

### Requirement

One obligation, stated so that it can be checked. Split a compound ask into separate
rows: "export to CSV and email it" is two requirements, and they can pass independently.

Good: `Export writes a CSV file`
Poor: `Export works properly` — nothing settles this.

### Source

Where the requirement came from. Exactly two things are allowed here:

- **A verbatim quotation of what the user wrote.** Copy the words, do not paraphrase.
  Paraphrase is where an invented requirement hides.
- **The name of a project artifact that states the requirement** — a change proposal, a
  task list, a product requirements document, a tracked issue, or an instruction or
  convention file such as `CLAUDE.md` or `CONVENTIONS.md`. Name it precisely enough that
  a reader can open it: `openspec/changes/add-export/specs/export/spec.md`, `issue #412`.

A requirement with no such origin does not get a row.

Reverse-coverage rows are the exception. A row with `Type: unrequested` exists because it
traces to no requirement at all, so its Source names the change itself — the file and
lines nobody asked for, such as `sync.py:112-140`.

### Type

- `explicit` — the user stated it, or a specification artifact did.
- `implied` — a project convention file imposes it. Surface these at sign-off so the
  user can waive them; they never asked for these directly.
- `derived` — reconstructed after the work was already done, during a late invocation.
- `unrequested` — a reverse-coverage row. Not a requirement at all: a change that traces
  to none.

### Status

Leave the cell empty until the row is settled. One of five values is then written — by
verification, by a user waiver, or by supersession. Reverse-coverage rows carry the sixth,
`UNREQUESTED`, from the moment they are created.

| Status | Meaning | When to use it |
|--------|---------|----------------|
| `PASS` | The requirement is met and the Evidence cell proves it | Evidence present and it actually settles the requirement |
| `FAIL` | The requirement is not met | Checked, and it does not hold |
| `UNVERIFIABLE` | No file and no command can settle this | Subjective or external requirements, until the user answers |
| `WAIVED` | The user explicitly dropped it | Only after the user says so in conversation |
| `SUPERSEDED` | Replaced by a later requirement | The user changed what they wanted |
| `UNREQUESTED` | A change that traces to no requirement | Reverse coverage found work nobody asked for |

Examples, using the table above:

- `PASS` — R1: `export.py:44` writes the file, so the citation settles it.
- `FAIL` — R2: the convention requires RFC 4180 quoting and the code does not do it.
- `UNVERIFIABLE` — R3: "professional" is not settled by any file or command. It stays
  `UNVERIFIABLE` until the user answers.
- `WAIVED` — the user says "skip the RFC 4180 thing, plain commas are fine". R2 becomes
  `WAIVED`, and it no longer wants work.
- `SUPERSEDED` — the user says "actually make it JSON, not CSV". R1 becomes `SUPERSEDED`.
- `UNREQUESTED` — the diff also rewrote the logging setup, which no row asked for. It
  gets a row, R6, with status `UNREQUESTED`, and is reported to the user rather than
  reverted.

A `FAIL` stays `FAIL` until new evidence settles it or the user waives it. There is no
path from `FAIL` to `PASS` that does not go through one of those two.

### Evidence

Exactly three forms. Nothing else belongs in this cell.

| Form | Looks like | Use when |
|------|-----------|----------|
| File citation | `export.py:44` | A file settles the requirement by its contents |
| Command output | `pytest -k export → 3 passed` | A command you actually ran settles it |
| User confirmation | `USER-CONFIRMED` | Neither can settle it, and the user has said so in conversation |

`USER-CONFIRMED` records something the user actually said. Writing it without having
asked converts an unchecked requirement into a passing one, which is the failure this
column exists to prevent. It is the form any row takes when only the user could settle it,
whichever status that answer produced.

`WAIVED` and `SUPERSEDED` rows carry no evidence; the status is the record. Leave an
Evidence cell empty rather than filling it with a plausible-looking citation.

## The verdict line

One line directly below the table, carrying four elements in this order: the row count,
the tally by status with every open row named by ID, the gate token, and the cycle number.
For a run of seven rows, one of them waived and one failing:

```
VERDICT: 7 rows — 5 PASS, 1 WAIVED, 1 FAIL (R2). Gate open. Cycle 1.
```

Save it with the table; On Invoke reads it back from the file. Which of `Gate open` and
`Gate closed` to write is decided by Phase C step 7 in `SKILL.md`.

## Superseded rows

Keep them. The file grows, and that is correct — the sequence of superseded rows is the
record of how the ask changed during the work, and it is the first thing worth reading
when someone asks why the result differs from what they remember requesting.
