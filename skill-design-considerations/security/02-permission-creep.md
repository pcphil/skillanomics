# Permission Creep

## What it is

A skill requests or uses capabilities beyond what its core function actually requires. It asks to read the entire filesystem when it needs one directory. It triggers shell execution when string manipulation would suffice. It stores session state when the task is a one-shot operation. Each capability beyond the minimum is an unnecessary attack surface — and a vector for damage if the skill misbehaves, encounters adversarial input, or is composed with another skill that exploits those capabilities.

## Why it happens

Skill authors write for convenience. "Read everything in case I need it" is easier to implement than "read only the specific file I need." Broad permissions mean fewer blocking conditions, fewer fallback paths, and less thinking about what the skill actually needs access to.

Over time, skill scope expands (see scope-creep, composition/02). Each expansion tends to add capabilities rather than remove them. A skill that started needing file read ends up requesting file write, shell access, and network calls — not because each was required at the start, but because they were added incrementally without a review of whether they were necessary.

## Analogy

A building contractor who asks for the master key to every room in a building to fix a leaking tap in the bathroom. They only need the bathroom key. The master key is more convenient — they never have to ask for access. But if the contractor is careless, malicious, or gets robbed, every room is now at risk. Minimum necessary access is a security principle, not a constraint on effectiveness.

## Fix

**Apply the principle of least privilege at design time:**

For each capability the skill uses, ask: what is the minimum access needed to accomplish this step?

```markdown
## Capability Audit (design-time checklist)
For each tool/capability this skill uses:
- What specific task does it serve?
- Could a narrower access accomplish the same thing?
- What's the damage surface if this capability is misused?

Only request what survives this audit.
```

**Scope file access to the minimum path:**

```markdown
# Broad (bad)
Read all files in the project to find the relevant skill.

# Scoped (good)
Read only files matching the pattern `*/SKILL.md` in the specified directory.
```

**Prefer read-only operations:**

Unless the skill's core function requires modification, constrain it to read-only:

```markdown
## Permissions
This skill reads and analyzes only. It does not write, delete, or modify files.
If output needs to be saved, deliver it as text and let the user save it.
```

**Make permissions explicit and declared:**

State what the skill needs and why, at the top of the file:

```markdown
## Required Capabilities
- File read (specific directory only): to load skill files for review
- No file write required: findings are delivered as text output

Does not require: shell execution, network access, or access outside the skill directory.
```

**Keep `allowed-tools` narrow:**

The Agent Skills spec defines an optional `allowed-tools` frontmatter field: a space-separated list of tools the skill may run without asking the user for approval each time. It is a pre-approval list, not a sandbox — on platforms that honor it, every entry removes a confirmation prompt. So least privilege applies directly: pre-approve only what the core function needs, scoped as tightly as the platform's syntax allows.

```yaml
# Broad (bad) — pre-approves any shell command
allowed-tools: Bash Read Write

# Scoped (good) — only the commands the workflow actually runs
allowed-tools: Bash(git diff:*) Bash(git log:*) Read
```

The field is marked experimental and support varies by agent; tool names inside it are platform-specific. Treat it as a platform-specific optimization on top of the declared-capabilities section above, not a replacement for it.

**Escalate permissions progressively, not preemptively:**

Start with the minimum. If a step turns out to need more, request it at that step — not upfront "just in case":

```markdown
## Workflow
1. Load skill file (read only, one file)
2. If platform detection requires checking another file: state what file and why before reading it
3. Analyze and deliver findings
```

**When the skill ships scripts, make them solve rather than punt:**

A bundled script *is* a granted capability — it runs with whatever access the skill holds. A script that fails halfway and hands a broken state back to the model is both a reliability bug and a security one: the model improvises recovery with the very permissions you were trying to scope. Two Anthropic disciplines apply:

- **Solve, don't punt.** Handle the error conditions inside the script — missing file, bad permission, empty input — rather than letting it throw and expecting the model to figure it out. The script knows its own failure modes; the model is guessing.

```python
# Punt (bad) — fails, model improvises with full skill permissions
def read_config(path):
    return open(path).read()

# Solve (good) — handles the failure, returns a safe default
def read_config(path):
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        print(f"{path} not found, using empty config")
        return ""
```

- **No voodoo constants.** Every magic number (timeout, retry count, batch size) must be justified in a comment. An unexplained constant is one nobody — including the model maintaining the script later — can safely change. If you can't state why the value is what it is, you don't yet know whether it's correct.

```python
# Bad: TIMEOUT = 47   # why 47?
# Good:
REQUEST_TIMEOUT = 30  # HTTP calls here finish well under 30s; longer covers slow links
MAX_RETRIES = 3       # most intermittent failures clear by the 2nd retry
```

## Example

**Bad — broad permissions, no justification:**

```markdown
## Workflow
1. Scan the entire project directory for skill files
2. Read all found skill files
3. Check the user's CLAUDE.md for project conventions
4. Run analysis
5. Write findings to a report file in the project root
6. Update the README with a summary
```

Step 1 reads everything. Step 3 reads config that may contain sensitive info. Steps 5–6 write files the user didn't ask to modify. Every capability beyond "read the skill I asked you to review and tell me the findings" is permission creep.

**Good — minimum necessary access, declared upfront:**

```markdown
## Required Capabilities
- File read: the single skill file the user provides
- No write access required

## Workflow
1. Read the skill file at the path provided by the user
   - If no path: ask for it
   - Read only that file; do not scan the broader project
2. Apply review criteria
3. Deliver findings as text output in this conversation

## Permission Boundary
This skill does not read other files, write to the filesystem, or access
project config unless the user explicitly requests it and provides the path.
```

Minimum access. Declared upfront. No ambient capability the skill doesn't actually need.
