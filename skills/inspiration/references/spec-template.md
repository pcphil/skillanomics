# Spec Template (Wrap phase)

Write accepted ideas in this format. Fill from the session only; leave out anything the user did not accept.

```markdown
# Inspiration Spec: <repo or project name>

**Project type:** <app | web | api | mixed>
**Focus:** <Polish | Opportunity | Both>
**Coverage:** scanned N of M areas, skipped X
**Date:** <YYYY-MM-DD>

## Context

- **Users and job:** <from Context grill>
- **Success signal:** <from Context grill>
- **Constraints:** <from Context grill>

## Polish

### <Idea title>
- **Theme:** <cluster name>
- **Impact / Effort:** <Low|Med|High> / <Low|Med|High>
- **Evidence:** `path/to/file:line`
- **What changes:** <1-3 sentences describing the improvement>
- **Why it helps:** <who gains and how>

(repeat per accepted Polish idea, ordered by impact then effort)

## Opportunities

### <Idea title>
- **Theme:** <cluster name>
- **Impact / Effort:** <Low|Med|High> / <Low|Med|High>
- **Anchor:** `path/to/file:line` or "user: <quote>"
- **Problem:** <one sentence>
- **User:** <who benefits first>
- **Success signal:** <observable change>
- **Smallest first slice:** <least we could build that tests the idea>

(repeat per accepted Opportunity, ordered by impact then effort)

## Deferred

- <idea>: <reason or revisit trigger>

## Rejected

- <idea>: <user's reason>

## Parked Bugs

- `path/to/file:line`: <one-line description>
```

Omit empty sections. Keep each idea under 10 lines.
