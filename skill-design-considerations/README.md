# Skill Design Considerations

A field guide to the failure modes that break agent skills — and how to avoid them.

Skills fail in predictable ways. Understanding why helps you write skills that hold up in real use, not just in demos. These considerations are organized into disciplines drawn from cognitive psychology, systems design, and engineering.

---

## Attention

How models allocate focus. Critical instructions get ignored not because they're wrong, but because of where they land in context.

| File | Problem |
|------|---------|
| [Lost in the Middle](attention/01-lost-in-the-middle.md) | Rules buried mid-file or mid-list get less attention than rules at the edges |
| [Over-Specification](attention/02-over-specification.md) | Too many rules → model satisfies few reliably |
| [Context Bloat](attention/03-context-bloat.md) | Loading too much at once crowds out the instructions that matter |
| [Instruction Hierarchy Blindness](attention/04-instruction-hierarchy-blindness.md) | Skill doesn't account for where it sits among system prompt, CLAUDE.md, other skills, and user messages — and often loses conflicts |
| [Stale Reference Loading](attention/05-stale-reference-loading.md) | Multi-phase skill loads wrong or outdated reference for the current step |

---

## Grounding

Keeping the model tethered to skill instructions as conversations evolve and pressure mounts.

| File | Problem |
|------|---------|
| [Negation Failure](grounding/01-negation-failure.md) | "Don't do X" leaves the right action undefined — positive framing is more reliable |
| [State Leakage](grounding/02-state-leakage.md) | Skill keeps influencing behavior after its task ends — including persona capture |
| [Instruction Conflict](grounding/03-instruction-conflict.md) | Skill instructions clash with system prompt, CLAUDE.md, or user messages |
| [Conversational Drift](grounding/04-conversational-drift.md) | Recent conversation gradually overrides skill instructions |
| [Termination Ambiguity](grounding/05-termination-ambiguity.md) | No clear "done" signal; model stays partially in skill mode |

---

## Robustness

Designing for the real world — where inputs are incomplete, platforms vary, and things go wrong.

| File | Problem |
|------|---------|
| [Trigger Pollution](robustness/01-trigger-pollution.md) | Vague description causes skill to fire for unrelated requests |
| [Happy-Path-Only Design](robustness/02-happy-path-only.md) | Skill only defines what to do when everything goes right |
| [Platform Capability Assumption](robustness/03-platform-capability-assumption.md) | Skill assumes tools or features not available on all platforms |
| [Version Assumption](robustness/04-version-assumption.md) | Skill relies on model capability that changes across versions or providers |
| [Context Compaction Fragility](robustness/05-context-compaction-fragility.md) | Skill state breaks when conversation is compressed/summarized by the platform |

---

## Composition

How skills behave as part of a larger system — with other skills, platforms, and growing scope.

| File | Problem |
|------|---------|
| [Skill Composition Blindness](composition/01-skill-composition-blindness.md) | Skill assumes it's the only active skill; breaks when others are loaded |
| [Scope Creep](composition/02-scope-creep.md) | Skill grows beyond its original purpose and loses focus |
| [Priority Inversion](composition/03-priority-inversion.md) | Lower-priority skill wins due to recency/proximity when two skills are active |

---

## Anatomy

Structural and architectural failure modes — what goes wrong when the skeleton of a skill is wrong, independent of content quality.

| File | Problem |
|------|---------|
| [Section Role Confusion](anatomy/01-section-role-confusion.md) | Content placed in the wrong section gets processed through the wrong cognitive lens |
| [Archetype Mismatch](anatomy/02-archetype-mismatch.md) | Skill uses the structural pattern of the wrong archetype for its purpose |
| [Missing On Invoke Contract](anatomy/03-missing-on-invoke-contract.md) | No initialization logic; model improvises entry differently each time |
| [Monolithic Workflow](anatomy/04-monolithic-workflow.md) | Flat undifferentiated steps where phases with different behaviors are needed |
| [Phantom Reference Architecture](anatomy/05-phantom-reference-architecture.md) | SKILL.md references files that don't exist or don't match expectations |

---

## Calibration

How skill instructions shape model confidence and uncertainty. Skills can push models toward overconfident output, fabrication, or accepting wrong premises.

| File | Problem |
|------|---------|
| [Overconfidence Bias](calibration/01-overconfidence-bias.md) | Skill instructs model to act without uncertainty acknowledgment |
| [Hallucination Amplification](calibration/02-hallucination-amplification.md) | Skill structure invites fabrication through open-ended enumeration patterns |
| [Assumption Propagation](calibration/03-assumption-propagation.md) | Model treats user's framing as fact; skill never prompts challenge |
| [Example Anchoring](calibration/04-example-anchoring.md) | Model copies incidental details from examples instead of extracting the principle |

---

## Interaction

The UX contract between skill and user. How the skill communicates state, handles failure, and calibrates output volume.

| File | Problem |
|------|---------|
| [Silent Failure](interaction/01-silent-failure.md) | Skill doesn't surface errors or explain why it stopped |
| [Mode Opacity](interaction/02-mode-opacity.md) | User can't tell what state or phase the skill is currently in |
| [Verbosity Miscalibration](interaction/03-verbosity-miscalibration.md) | Output volume is mismatched to the task — too long or too short |
| [Feedback Loop Absence](interaction/04-feedback-loop-absence.md) | No defined revision mechanism — model regenerates everything or applies feedback inconsistently |

---

## Security

Skills handling untrusted data or elevated capabilities.

| File | Problem |
|------|---------|
| [Prompt Injection Blindness](security/01-prompt-injection-blindness.md) | Skill passes tool results into reasoning without treating them as untrusted |
| [Permission Creep](security/02-permission-creep.md) | Skill requests more capabilities than its core function requires |

---

## Principles

Positive design guidance — what good skill design looks like, not just what to avoid.

| File | Principle |
|------|-----------|
| [Minimal Surface Area](principles/01-minimal-surface-area.md) | Define the smallest behavioral contract that accomplishes the skill's purpose |
| [Fail Visible](principles/02-fail-visible.md) | Surface ambiguity and failure explicitly — never guess or fail silently |
| [Adversarial Coverage](principles/03-adversarial-coverage.md) | Test against off-happy-path inputs before shipping |
| [Positive Constraint Grammar](principles/04-positive-constraint-grammar.md) | Express all constraints as actions to take, not things to avoid |
| [Idempotent Output](principles/05-idempotent-output.md) | Same input should produce structurally consistent output across runs |
| [Explicit State Management](principles/06-explicit-state-management.md) | All workflow state visible in responses — never carried implicitly in context |

---

## How to Use This

**Authoring a new skill:** Read the Principles section first, then scan the discipline most relevant to your skill type.

**Diagnosing a broken skill:** Start with the symptoms. Each failure mode document lists observable symptoms — match your symptoms to the discipline, then read the specific failure mode.

**Structure of failure mode files:** what it is → why it happens → analogy → symptoms → fix → example

**Structure of principles files:** what it is → why it matters → how to apply → example
