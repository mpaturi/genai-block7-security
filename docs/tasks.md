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

**Every phase ends the same way:** push the branch, open a PR against
`main` (or, for the one dependent case, against the branch it depends
on), and stop. Do not merge — that's the reviewer's call, per house rules.

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
   spec describes? If not, note the drift before continuing.
3. Locate every call site that can fail transiently (LLM calls, Neo4j
   calls) and add exponential backoff with a capped number of attempts.
4. Add exception classification so permanent failures (bad input) fail
   immediately instead of retrying.
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
4. For each test case, assert the parsed output stays within expected
   type/domain/range, and that anything suspicious gets surfaced in
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
4. Add field-layer trimming so citations only carry what the answer
   needs, not the full raw note.
5. Write a regression test proving the sanitization holds structurally
   (feed it a deliberately malicious note, confirm the dangerous part
   never survives).
6. Check Block 4's existing eval harness assertions on citation content
   against the now-trimmed citations — this is the open follow-up
   already flagged in the spec. Fix or update those assertions if they
   broke.
7. Commit sanitization and trimming as one logical change (same code
   path, same purpose).
8. Push, open PR against `main`.

## Phase: Block 6 — state validation

1. Branch off `main` in `genai-block6-multiagent`.
2. Verify: is `MultiAgentState` still the mutable `TypedDict` the spec
   describes, passed between the same nodes?
3. Add schema/type validation at each node boundary where state gets
   written.
4. On a failed validation, log a clear warning and mark the entry as
   suspect rather than silently propagating it into reconciliation.
5. Write a test that deliberately writes a malformed value at a node
   boundary and confirms it's caught and logged, not trusted.
6. Commit.
7. Push, open PR against `main`.

## Phase: Block 6 — Cohort agent injection test and query-size visibility

1. Branch off `main` in `genai-block6-multiagent`.
2. Verify: is the Cohort agent's query still the single parameterized,
   read-only Cypher query the spec describes?
3. Write a dedicated injection test for that query, matching Block 5's
   existing rigor — assert on the actual query structure and parameter
   dict sent to the driver, not just "no crash."
4. Add logging of row count and runtime for each query execution.
5. Add a configurable soft-alert threshold that flags unusually large
   results in tracing.
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