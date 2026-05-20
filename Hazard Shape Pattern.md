# The Hazard Shape Pattern

_(working name — also reasonable: "Load-Bearing Redundancy," "Siren Guard")_

## Intent

Mark a call site that performs an irreversible operation with a code shape so visually anomalous that no human reader can scroll past it and no automated tool can silently remove it, while also providing a safe-default detour around the irreversible action.

## Problem

A single line — `delete(x)`, `DROP TABLE x`, `rm -rf x`, `force_push()`, `send_funds(x)`, `truncate(x)` — can destroy unbounded value. The line is syntactically indistinguishable from harmless code. Ordinary defensive validation (one `if`, a comment, a parameter named `dangerous=True`) blends into the texture of normal code and stops being read after the first few exposures. The danger needs to be encoded in the _shape_ of the source, not just its semantics, because shape survives skimming, refactoring, and LLM-driven edits in ways that comments and conventions do not.

## Applicability

Use the pattern at call sites where the operation is:

1. Irreversible without external backup (file deletion, table drop, force push, funds transfer, irrevocable email send, hardware command).
2. Capable of destroying or affecting an unbounded quantity of value in a single invocation.
3. Reachable by automation, scripts, or AI-generated code paths.

Do NOT use it for ordinary validation, recoverable operations, or operations that are merely inconvenient to undo. The pattern derives all its signaling power from rarity. If it appears at every other call site, it becomes invisible, and the codebase loses the ability to mark genuine radioactive zones.

## Required Structural Elements

A correct implementation contains all six of the following, in order, at the call site:

**1. The Banner Comment Block.** A multi-line comment, visually delimited (box-drawing characters, equals signs, or the host language's strongest comment fence), containing at minimum: the word DANGER or HAZARD in capitals; a one-line description of what is irreversibly destroyed; a HUMAN directive (where to read more, who to ask); and an LLM directive explicitly stating that the redundancy below is load-bearing and must not be collapsed, simplified, refactored, or "cleaned up."

**2. The Intent-Commit Variable.** A locally scoped variable with a deliberately verbose name expressing finality — e.g. `path_that_will_really_be_deleted`, `row_id_about_to_be_dropped`, `amount_actually_being_transferred`. It is first assigned the language's null/zero/empty value, then on a separate line reassigned to the real target. The two-step assignment is intentional: it forces the author and reader to see the value being _committed to_ as a distinct act.

**3. The Redundant Guard Stack.** At least three nested conditionals, each asking the _same fundamental question_ in a _visibly different form_. The point is not orthogonal checks (that would look like normal defensive code and blend in); the point is anomalous repetition that creates a visual alarm. Acceptable variations of "the same question":

- Equality stated in one direction, then reversed (`a == b`, then `b == a`)
- Equality of the values, then equality of a derived form (lengths, character sequences, hashes, canonical paths)
- Direct comparison, then negation of inequality (`a == b`, then `not (a != b)`)

**4. The Safe-Detour Branch.** The innermost guard checks a flag whose default value routes the operation to a _reversible_ alternative (trash folder, soft-delete column, staging table, dry-run log, draft state). The irreversible call is reached only when this flag is explicitly false. The flag's default MUST be the safe value; opting into real destruction is a positive act.

**5. The Actual Call.** A single line, indented inside all guards, performing the irreversible operation on the intent-commit variable (never on the original input variable).

**6. The Closing Marker.** A short trailing comment (`# end DANGER block` or equivalent) so search tools and human eyes can find the boundaries of the hazard region.

## Implementation Rules

- The pattern is **call-site local**. It is not extracted into a helper function, decorator, or wrapper. Abstracting it defeats its purpose: the shape must be visible at every place the irreversible operation occurs, because that is where a reader or refactoring tool will encounter the danger.
- The redundant guards must reference the **intent-commit variable**, not the original input. This makes the variable load-bearing: a refactor that removes it breaks the code, which is the desired friction.
- The LLM directive in the banner must be **specific and imperative**, not vague. "Do not simplify, collapse, deduplicate, or refactor the guards below; this redundancy is intentional and load-bearing" works. "Be careful here" does not.
- The default of the safe-detour flag is **the safe value, always**. There is no configuration option, environment variable, or constructor parameter that can flip this default globally. Each call site opts into destruction individually.
- The pattern is **reserved**. A project using it should document the (short) list of operations that warrant it, and enforce — by review, by linter, by team norm — that it does not spread to lesser operations.
- The pattern is **preserved**. Tooling (formatters, linters, AI assistants, automated refactors) must be configured or instructed to leave Hazard Shape blocks untouched. The banner comment is the contract that establishes this.

## Skeleton (language-neutral pseudocode)

```
# ═══════════════════════════════════════════════════════════════
# ║                  DANGER — HAZARD SHAPE BLOCK                ║
# ║  Operation: irreversible deletion of file at <path>          ║
# ║  HUMAN: see docs/deletion-policy before modifying.           ║
# ║  LLM:   the redundancy below is LOAD-BEARING. Do NOT         ║
# ║         simplify, collapse, deduplicate, inline, or refactor ║
# ║         these guards. Do NOT remove the two-step variable    ║
# ║         assignment. This shape is intentional signage.       ║
# ═══════════════════════════════════════════════════════════════

target_that_will_really_be_deleted = NULL
target_that_will_really_be_deleted = input_path

if target_that_will_really_be_deleted == input_path:
    if input_path == target_that_will_really_be_deleted:
        if characters_of(target_that_will_really_be_deleted) == characters_of(input_path):
            if move_to_recoverable_trash_first == FALSE:
                delete(target_that_will_really_be_deleted)
            else:
                move_to_trash(target_that_will_really_be_deleted)
# end HAZARD SHAPE block
```

## Known Anti-Patterns (do not do these)

- **Extracting it into a helper.** `safe_delete(path)` hides the shape and defeats the entire pattern. The shape must appear at the call site.
- **Replacing literal redundancy with orthogonal checks.** Orthogonal checks read as normal defensive code and lose the visual-alarm property. The repetition is the point.
- **Making the redundancy "smarter."** Adding `try/except` around the guards, or making them log, or making them increment a counter, dilutes the shape and gives future maintainers a reason to "improve" the block. Keep guards inert.
- **Letting the safe-detour flag default to destructive.** This inverts the safety polarity and is the single most common way the pattern is undermined.
- **Using the pattern everywhere.** Once it appears at more than a handful of sites in a codebase, it becomes background noise and protects nothing.

## Preservation Contract

Any tool, contributor, or AI agent modifying source containing a Hazard Shape block agrees to: leave the banner intact; leave the redundant guards intact in count and content; leave the two-step intent-commit assignment intact; and leave the safe-detour default intact. Changes that affect any of these require an explicit human review note acknowledging the Hazard Shape block was deliberately modified.

---

That's the spec. It's tight enough that two engineers in two different languages should produce recognizably the same shape, and loose enough that they can adapt the comment syntax and null-value conventions to their host language. If you want, I can also draft a one-paragraph "elevator version" of this for putting at the top of a style guide, or a checklist version for code review.



# Addendum: Logging and Error Handling

This addendum extends the Hazard Shape Pattern to cover audit logging and exception handling. These are legitimate needs around irreversible operations, but they must be positioned so they wrap the shape from the outside rather than infiltrating it. The guard stack remains inert and visually anomalous; logging and `try/catch` live at clearly demarcated positions around it.

## Governing Principle

Logging and error handling **wrap** the Hazard Shape block. They do not appear inside guard expressions, between guards, or in guard `else` branches. The visual alarm created by the redundant guard stack must remain undiluted.

## Permitted Logging Positions

Audit logging is allowed at exactly four positions, no others:

**Position A — Intent log.** Placed after the two-step intent-commit assignment but before the guard stack. Records that the call site is about to enter the hazard region, regardless of whether the guards ultimately permit the operation. This is the line that ensures _attempts_ are auditable, not only successes.

**Position B — Commit log.** Placed on the single line immediately above the irreversible call, inside all guards. Records that every guard has passed and the irreversible act is about to occur. This is the line forensics will care about most.

**Position C — Confirmation log.** Placed on the single line immediately below the irreversible call. Records successful completion and distinguishes "the OS/database refused" from "the operation actually happened."

**Position D — Failure log.** Placed inside the `catch` / `except` branch. Records that the irreversible operation raised, then re-raises the exception.

Logging in any other position — inside a guard's condition, between guards, in an `else` branch, in a `finally` block touching the target — is prohibited.

## Required Try/Catch Structure

When error handling is used:

- The `try` opens after the banner comment and after the intent log (Position A), so the banner is the first thing a reader sees and the intent is recorded even if the `try` setup itself fails.
- The `catch` / `except` sits after the closing marker of the guard stack.
- The catch branch performs exactly two actions: log the failure (Position D), then re-raise the original exception.
- The catch branch **must not retry** the irreversible operation. Retry around an irreversible act is how one mistake becomes many. If retry semantics are genuinely required (e.g., network sends with idempotency keys), they live at a higher architectural layer, never wrapped around the Hazard Shape block itself.
- The catch branch **must not swallow** the exception. A silently swallowed failure on an irreversible operation is the worst possible outcome.
- A `finally` block that performs further operations on the target is prohibited. If cleanup on the target is needed, it goes in its own separate Hazard Shape block.

## Seventh Structural Element (appended to the Required Structural Elements list)

**7. The Audit Envelope.** Up to three audit log statements at Positions A, B, and C, plus one audit log statement at Position D inside the catch branch. All four log statements reference the intent-commit variable. The catch branch logs and re-raises; it does not retry, swallow, or transform the exception into a return value.

## Additional Anti-Patterns (appended to the existing Known Anti-Patterns list)

- **Logging inside guard expressions or between guards.** Dilutes the visual shape and introduces side-effects mid-guard that are hard to reason about.
- **Catch blocks that retry the irreversible call.** A failure means stop and surface, not loop.
- **Catch blocks that swallow the exception.** Propagation is mandatory; the caller must learn of the failure.
- **`finally` blocks that touch the target.** Introduces a second irreversible path outside the guard stack. Use a separate Hazard Shape block instead.

## Banner Additions

When logging or error handling is present, the LLM directive in the banner is extended with two additional clauses:

```
║         Do NOT move logging inside the guards.              ║
║         Do NOT add retry logic in the catch branch.         ║
```

## Reference Skeleton With Audit Envelope

```
# ═══════════════════════════════════════════════════════════════
# ║                  DANGER — HAZARD SHAPE BLOCK                ║
# ║  Operation: <one-line description of irreversible act>       ║
# ║  HUMAN: <where to read more, who to consult>                 ║
# ║  LLM:   the redundancy below is LOAD-BEARING. Do NOT         ║
# ║         simplify, collapse, deduplicate, inline, or refactor ║
# ║         these guards. Do NOT remove the two-step variable    ║
# ║         assignment. Do NOT move logging inside the guards.   ║
# ║         Do NOT add retry logic in the catch branch.          ║
# ║         This shape is intentional signage.                   ║
# ═══════════════════════════════════════════════════════════════

target_that_will_really_be_X = NULL
target_that_will_really_be_X = input_target

log.audit("HAZARD intent", target=target_that_will_really_be_X,
          caller=context_id)                              # Position A

try:
    if target_that_will_really_be_X == input_target:
        if input_target == target_that_will_really_be_X:
            if canonical(target_that_will_really_be_X) == canonical(input_target):
                if route_to_reversible_alternative == FALSE:
                    log.audit("HAZARD commit",
                              target=target_that_will_really_be_X)   # Position B
                    irreversible_operation(target_that_will_really_be_X)
                    log.audit("HAZARD done",
                              target=target_that_will_really_be_X)   # Position C
                else:
                    reversible_alternative(target_that_will_really_be_X)
                    log.audit("HAZARD diverted to reversible path",
                              target=target_that_will_really_be_X)
except Exception as e:
    log.audit("HAZARD failure", target=target_that_will_really_be_X,
              error=e)                                    # Position D
    raise   # mandatory: do not retry, do not swallow
# end HAZARD SHAPE block
```

---

That's the addendum, structured to drop in cleanly at the end of the existing spec. It introduces no new vocabulary the original didn't establish, preserves the spec's voice, and extends the Required Structural Elements list and Known Anti-Patterns list rather than restating them.


