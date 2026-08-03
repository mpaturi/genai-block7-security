# Block 7 — Security & Hardening: Tasks

Concrete, ordered checklists for each phase in `docs/plan.md`. Nothing
here has been committed anywhere — this is a draft for review alongside
the spec and plan.

**Branching rule for this block:** independent phases branch directly off
their repo's current `main`. Only phases with a real dependency on
another phase chain off that phase's branch instead — the one case here
is the Block 6 dependency pin bump, which branches off `main` only after
the Block 5 retry-hardening phase has merged into it. Every other phase
below branches off `main` regardless of what else is in flight.

**Verification rule for this block:** every phase's step 2 checks the
spec's claims against the real code before building anything. If what's
changed is cosmetic — a rename, a moved file, same actual behavior and
data shape — note it and keep going. If what's changed is structural —
different fields, a described flow that no longer exists, something this
phase depends on being removed or reworked — stop, fix `docs/spec.md`
first, and only resume once the spec matches reality again.

**Every phase ends the same way:** push the branch, open a PR against
`main` (or, for the one dependent case, against the branch it depends
on), and stop. Do not merge — merging is gated on external approval, per
house rules.

## Phase: Block 7 — spec and plan

1. Confirm `docs/spec.md` and `docs/plan.md` are in their final,
   approved form (this review).
2. Branch off `main` in `genai-block7-security`.
3. Commit `docs/spec.md`.
4. Commit `docs/plan.md` as a separate commit on the same branch — two
   commits, one PR, since they're sequential planning documents, not
   unrelated changes.
5. Push, open PR against `main`.

## Phase: Block 5 — retry/backoff hardening

1. Branch off `main` in `genai-block5-agent`.
2. Verify: does `main` still call the LLM and Neo4j the same way the
   spec describes? Apply the verification rule above if not.
3. Locate every call site that can fail transiently (LLM calls, Neo4j
   calls) and add exponential backoff, using the retry count and timing
   defined in the spec's LLM10 section — confirm those numbers against
   Block 6's actual Phase 8 code first, and update the spec if they
   differ.
4. Add exception classification so permanent failures (bad input) fail
   immediately instead of retrying, using the retryable/non-retryable
   error list defined in the same spec section.
5. Write a test that simulates a transient failure and asserts the wait
   grows between attempts and the system gives up after the cap.
6. Run the test suite, confirm everything passes, not just the new test.
7. Commit the retry/backoff logic and its test as one logical change
   (or two commits if the logic and test are large enough to separate
   cleanly — judgment call at implementation time).
8. Push, open PR against `main`.

## Phase: Block 5 — direct-injection test suite

1. Branch off `main` in `genai-block5-agent`.
2. Verify: does the query-parsing step still take a raw natural-language
   question and return the same structured fields the spec describes?
3. Write adversarial test cases: instruction-override attempts ("ignore
   prior instructions..."), attempts to extract the system prompt,
   attempts to steer parsed fields toward attacker-chosen values.
4. For each test case, assert the parsed output against the plausibility
   rule in the spec's LLM01 section (query the Neo4j graph once for its
   real condition/lab/drug values, cache them, check parsed values
   against that list), and that anything that fails gets surfaced in
   tracing rather than silently accepted.
5. Run the suite, confirm results match what the spec commits to
   (flagged, not silently blocked with false certainty).
6. Commit the test suite.
7. Push, open PR against `main`.

## Phase: Block 6 — citation hardening

1. Branch off `main` in `genai-block6-multiagent`.
2. Verify: is `chunk_text` still flowing into `MultiAgentAnswer.citations`
   the way the spec describes, and does it still never reach an LLM
   prompt anywhere in the current code?
3. Add the sanitization step where citations are constructed: strip or
   neutralize control sequences and instruction-like patterns.
4. Add field-layer trimming using the sentence-level keyword-containment
   rule defined in the spec's LLM02 section — not a freeform judgment
   call at implementation time.
5. Write a unit-level regression test proving the sanitization function
   holds structurally (feed it a deliberately malicious string directly,
   confirm the dangerous part never survives).
6. Write a second, end-to-end test: plant an injection attempt inside a
   seed patient note, run it through the real pipeline (Block 1's note
   generation, Block 3's storage, Block 4's retrieval, Block 6's citation
   construction), and confirm it's neutralized by the time it reaches
   `MultiAgentAnswer.citations`. This is the stronger proof — it tests
   the real system, not an isolated function.
7. Check Block 4's existing eval harness assertions on citation content
   against the now-trimmed citations — this is the open follow-up
   already flagged in the spec. Fix or update those assertions if they
   broke.
8. Commit sanitization and trimming as one logical change (same code
   path, same purpose), and the two tests as a separate commit.
9. Push, open PR against `main`.

## Phase: Block 6 — state validation

1. Branch off `main` in `genai-block6-multiagent`.
2. Verify: is `MultiAgentState` still the mutable `TypedDict` the spec
   describes, passed between the same nodes, and does
   `reconcile_node_safe` still call `_reconcile_error_answer` unguarded
   inside its except block, the way this spec now describes?
3. Add schema/type validation at each node boundary where state gets
   written.
4. On a failed validation, log a clear warning and mark the entry as
   suspect rather than silently propagating it into reconciliation.
5. Wrap `reconcile_node_safe`'s call to `_reconcile_error_answer` in its
   own try/except, falling back to a fixed literal `MultiAgentAnswer`
   with no computed fields if that helper itself raises.
6. Write a test that deliberately writes a malformed value at a node
   boundary and confirms it's caught and logged, not trusted.
7. Write a second test that forces `_reconcile_error_answer` to raise and
   confirms the system still returns a valid answer instead of the
   exception escaping.
8. Commit the state validation and the reconciliation error-handling fix
   as separate commits — two different concerns sharing one phase.
9. Push, open PR against `main`.

## Phase: Block 6 — Cohort agent injection test and query-size visibility

1. Branch off `main` in `genai-block6-multiagent`.
2. Verify: is the Cohort agent's query still the single parameterized,
   read-only Cypher query the spec describes?
3. Write a dedicated injection test for that query, matching Block 5's
   existing rigor — assert on the actual query structure and parameter
   dict sent to the driver, not just "no crash."
4. Add logging of row count and runtime for each query execution.
5. Add the soft-alert threshold defined in the spec's LLM10 section
   (500 patients or 25% of total population, whichever is smaller, or
   5+ second runtime) — treat these as defaults to tune once the real
   dataset size is confirmed, not fixed forever.
6. Confirm this new logging doesn't touch or duplicate Block 5's
   existing LLM cost/token tracking — different signal, different code
   path.
7. Commit the injection test and the logging as two separate commits —
   they're two different concerns sharing one code area, not one
   change.
8. Push, open PR against `main`.

## Phase: Block 6 — dependency pin bump

1. Confirm the Block 5 retry-hardening phase has actually merged first
   — this phase doesn't start until that's true.
2. Branch off `main` in `genai-block6-multiagent` (which now includes
   whatever was merged before this point).
3. Update `requirements.txt` to reference Block 5's new commit.
4. Run Block 6's test suite to confirm nothing broke from the bump.
5. Commit — this is a one-line, single-purpose change.
6. Push, open PR against `main`.

## Phase: Block 7 — SECURITY.md

1. Confirm every phase above has merged, not just opened as a PR.
2. Branch off `main` in `genai-block7-security`.
3. Write the RBAC section: the Community Edition limitation, stated
   plainly, plus the named Enterprise/Aura migration path.
4. Write the pinning policy section, matching what's now actually
   implemented across the repos.
5. Write one status line per risk (all five from the spec), each
   pointing at the specific test that proves its blocked/flagged claim
   — link to the actual test file and test name, not a vague reference.
6. Commit.
7. Push, open PR against `main`.
