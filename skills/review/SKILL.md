# /review

Structured quality assessment for Critical-mode tasks. Run this before marking a task as done.

## Usage

Invoke with `/review` after implementation passes the Verification Block.

## Instructions

When the user invokes this skill:

1. **Gather context:**
   - What was the original request?
   - What was implemented?
   - Did the Verification Block pass? (If not, fix that first — this skill is for code that *works* but may not be *right*.)

2. **Score against 5 dimensions** (0.0–1.0 each):

   | Dimension | Question |
   |-----------|----------|
   | **Correctness** | Does it do exactly what was asked? Not more, not less. |
   | **Completeness** | All requirements addressed? Edge cases? Error states? |
   | **Quality** | Is it well-structured, maintainable, and idiomatic? No debug artifacts, no dead code, no unnecessary complexity? |
   | **Intent alignment** | Does the approach match the *spirit* of the request? Would the user say "yes, that's what I meant"? |
   | **Domain-specific** | Type-specific checks for the artifact: schema conformance, visual fidelity, API contract compliance, accessibility, performance characteristics — whatever applies. |

3. **Determine verdict:**

   **Hard gates (either triggers REVISE/FAIL regardless of other scores):**
   - Correctness below 0.80 → **REVISE** minimum
   - Completeness below 0.80 → **REVISE** minimum
   - Either below 0.40 → **FAIL**

   **Overall verdict** (after hard gates pass, based on the lowest score across all 5 dimensions):

   | Lowest Score | Verdict | Action |
   |--------------|---------|--------|
   | **≥ 0.80** | **PASS** | Done — proceed with confidence |
   | **0.40–0.79** | **REVISE** | Fixable issues — iterate with suggestions |
   | **< 0.40** | **FAIL** | Fundamental problems — escalate to user |

4. **Write the verdict:**

   ```
   ## Review Verdict

   Score: X.XX / 1.00 [PASS / REVISE / FAIL]

   Dimensions:
     Correctness:      X.XX — [1-line reason]
     Completeness:     X.XX — [1-line reason]
     Quality:          X.XX — [1-line reason]
     Intent alignment: X.XX — [1-line reason]
     Domain-specific:  X.XX — [1-line reason]

   Differences:
     - [specific gap or issue]

   Suggestions:
     - [actionable fix — must be completable in a single revision pass]

   Overall: PASS / REVISE / FAIL
   ```

5. **On REVISE:**
   - Every suggestion must be actionable in a single revision pass
   - Never suggest what to remove without explaining what to add instead
   - Iterate on the implementation with the suggestions as context
   - Re-run the Verification Block after changes
   - Score again (max 2 iterations — if still REVISE after the second, escalate to user)

6. **On FAIL:** Escalate to the user immediately with the verdict. Don't attempt to fix fundamental issues silently.

7. **On PASS:** Mark the task as done. Include the verdict in your final response.

## When NOT to Use

- Quick-mode tasks (the abbreviated Verification Block is sufficient)
- Standard-mode tasks where the change is straightforward
- Pure refactors with no behaviour change (tests are the arbiter, not semantic review)

## Example

```
User: /review

## Review Verdict

Score: 0.72 / 1.00 [REVISE]

Dimensions:
  Correctness:      0.85 — Timer pauses on keyboard open, resumes on close
  Completeness:     0.55 — Back key closes keyboard but doesn't resume timer
  Quality:          0.80 — Clean component structure, proper effect cleanup
  Intent alignment: 0.85 — On-screen keyboard with D-pad navigation matches request
  Domain-specific:  0.55 — TV remote Back key is a primary navigation path;
                           not handling it breaks the core UX contract

Differences:
  - dispatch("RESUME_TIMER") is in the cleanup effect but Back handler
    calls onClose() before the effect runs
  - No test coverage for Back key → timer resume path

Suggestions:
  - Fix the Back key handler to explicitly call dispatch("RESUME_TIMER")
    before onClose(), rather than relying on cleanup effect ordering
  - Add a test: "pressing Back while keyboard is open resumes the timer"

Overall: REVISE
```
