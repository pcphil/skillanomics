# Ambiguity triage

This file defines the three triage labels and how to phrase the question round. `SKILL.md`
names the labels in Phase A and defers the rest here.

Every captured requirement gets one of three labels before work starts. The label decides
whether you pause.

## The test

Read the requirement and ask: **how many ways can this be read, and do those readings
lead to different work?**

| Label | Readings | Do they diverge? | What you do |
|-------|----------|------------------|-------------|
| `CLEAR` | One | — | Proceed |
| `ASSUMED` | Several | They converge on the same work | Proceed, and state the assumption |
| `AMBIGUOUS` | Several | They lead to materially different work | Pause and ask |

The pivot is not "am I certain" — it is "would being wrong change what I build". A
requirement you are unsure about but where every reading produces the same code is
`ASSUMED`, not `AMBIGUOUS`. Reserve the pause for the cases where a wrong guess wastes
the work.

## Worked examples

### `CLEAR`

> "Add a `--dry-run` flag that prints what would happen without writing anything."

One reading. The flag exists, it prints, it does not write. Nothing to ask.

Row: `Type: explicit`, triage `CLEAR`, proceed.

### `ASSUMED`

> "Log it when the sync fails."

Several readings — which log level, which destination, how much of the error. But every
reading produces the same shape of code: a log call on the failure path, using whatever
logging the project already uses. Picking wrong costs one line to change.

Row: triage `ASSUMED`. Proceed, and say so:

> Proceeding on the assumption that "log it" means an error-level entry through the
> project's existing logger, including the exception message.

### `AMBIGUOUS`

> "Make the export faster."

Readings diverge sharply. Faster could mean: stream rows instead of building the whole
file in memory; add an index to the query behind it; parallelise across chunks; or cache
results between runs. These are different pieces of work, days apart in effort, with
different risk. Guessing wrong means the work is wasted.

Row: triage `AMBIGUOUS`. Pause and ask.

### A second `AMBIGUOUS`, for contrast

> "It should handle bad input gracefully."

Diverging readings: reject the row and continue the run; reject the whole run; substitute
a default; or surface a prompt. Each produces different observable behavior, and the user
will notice which one you picked.

## Asking about ambiguities

Phase A step 5 puts every `AMBIGUOUS` row into a single message. What follows is the shape
that message takes: number the questions and give each a recommendation, so a user who
agrees can answer with one word.

Shape:

```
Three requirements need a decision before I start.

Q1 — R4, "make the export faster": which bottleneck?
  (a) Stream rows rather than building the file in memory
  (b) Index the query behind it
  (c) Parallelise across chunks
  → Recommend (a) — memory is the constraint at the row counts in the fixtures.

Q2 — R6, "handle bad input gracefully": what should a bad row do?
  (a) Skip the row, log it, continue
  (b) Abort the whole run
  → Recommend (a) — matches how the importer already behaves.

Q3 — R7, "keep the old format working": for how long?
  (a) Indefinitely, both formats supported
  (b) One release, then removed
  → Recommend (b) — (a) means maintaining two paths forever.
```

Why the rule is shaped this way:

- **Why one round.** Holding some ambiguities back for a later turn spends the user's
  attention twice and stalls the work in between.
- **Why all of them.** When triage produces a long list, the list is long. Trimming it to
  look considerate drops a real ambiguity, and the dropped one becomes the mistake this
  skill was invoked to prevent.
- **Why a recommendation on each.** The user can accept the lot in one word, or correct
  only the ones you got wrong.

## Re-triage during the work

A row appended mid-work faces the same test as one captured in Phase A: how many ways can
it be read, and do those readings diverge? Nothing about arriving late changes the answer,
and nothing about being mid-work makes a divergent reading cheaper to discover at the gate.

## Recording the outcome

- `CLEAR` rows need no annotation.
- `ASSUMED` rows carry the assumption stated in conversation at the time you proceed, so
  it is on the record before the work rather than after.
- `AMBIGUOUS` rows stop being ambiguous once the rewrite Phase A step 5 calls for has been
  made: the requirement text says what the user chose, and the Source is the answer they
  gave. A reader can then see from the row alone why it reads the way it does.
