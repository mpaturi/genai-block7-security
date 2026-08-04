# Block 7 — Security & Hardening: Implementation Plan

This plan turns `docs/spec.md`'s risks and decisions into phases. It
deliberately does not pin exact file paths or line-level detail — those
get confirmed against the real, merged code at the start of each phase,
not guessed at here. Nothing in this document has been committed anywhere;
it's a draft for review alongside the spec.

## 1. Working from current main

Every phase below starts the same way: confirm the spec's code-level
claims — how citations get built, the shape of `MultiAgentState`, the
Cohort agent's query, what Block 5's answer-writing call receives — still
match what's actually in `main` for that repo, before writing anything
against it. This is a five-minute sanity check, not a redo, and it applies
to each phase individually, not just once at the start. Cosmetic drift
(a rename, a moved file) just gets noted; structural drift (different
fields, a described flow that no longer exists) means stopping to fix
the spec first — see `docs/tasks.md` for the exact rule.

## 2. Which repo each fix lands in

**`genai-block7-security`** (no runtime code): `docs/spec.md`,
`docs/plan.md` (this document), and `docs/tasks.md` as one combined
planning PR, and `SECURITY.md` as a separate, later PR. This repo
documents the threat model; it doesn't own any of the application code
being hardened.

**`genai-block6-multiagent`**: citation sanitization and field-minimization
(LLM01 indirect, LLM02 field layer — same code path, one phase), state
validation at `MultiAgentState` boundaries (LLM06 tampering), the Cohort
agent's dedicated injection test plus query-size/runtime logging (LLM06
injection, LLM10 exhaustive query — same code area, one phase), and a
small dependency pin bump (its own phase). These are Block 6's own
objects; the fixes live where the objects live.

**`genai-block5-agent`**: retry/backoff hardening (LLM10 retry gap) and
the direct-injection test suite (LLM01 direct). Both target Block 5's own
agent code — the retry logic and the query-parsing step are Block 5's,
not Block 6's, even though Block 6 depends on them. Tests live with the
code they test, same as Block 5 and Block 6's existing parameterization
tests already do.

**`genai-block4-rag-eval`**: chunk-text sanitization before it reaches
`generate_answer`'s prompt (LLM01 Path B), a `max_length` cap on
`QueryRequest`'s four free-text fields (found during the same review),
and splitting the Pinecone API key into a query-scoped and a write-scoped
key (LLM08). Block 4 wasn't previously in Block 7's footprint; it enters
now because this is where the live indirect-injection exposure and the
key-scoping gap actually live, not because scope grew for its own sake.

## 3. Phases

Each phase is one branch, one focused PR, per house rules. Phase numbers
aren't assigned here — each repo continues its own existing phase
sequence. Assign the real number when the branch is actually cut, not
now.

**Block 7 — spec, plan, and tasks.** `docs/spec.md`, `docs/plan.md`, and
`docs/tasks.md`, combined into one PR as three separate commits, since
all three are pre-code planning documents produced back to back. No
application code.

**Block 5 — retry/backoff hardening.** Ports Block 6's Phase 8 pattern
(exponential backoff, exception classification) into Block 5's tool-calling
code, using the specific retry count, backoff timing, and retryable-error
list defined in the spec's LLM10 section (confirm those numbers against
Block 6's actual Phase 8 code first). Proves itself with tests that
simulate a transient failure and assert the wait grows between attempts
and the system gives up instead of retrying forever. Satisfies LLM10's
"retry gap: blocked" target.

**Block 4 — chunk sanitization and key scoping.** Neutralizes `chunk_text`
before it reaches `generate_answer`'s prompt (`generate.py`) — either at
ingestion into Pinecone or immediately before the prompt is built,
decided at this phase's start — and adds a `max_length` cap to
`QueryRequest`'s four free-text fields (`question`, `condition`, `drug`,
`lab`). Also splits Block 4's Pinecone API key: a new
`DataPlaneViewer`-scoped key for the query path (`retrieve.py`,
`api.py`), the existing full-access key kept only for the write path
(`create_index.py`, `ingest.py`, `verify.py`) — the new key created
manually in the Pinecone console first, per the spec's open follow-up.
Proves itself three ways: a test asserting the prompt Block 4 builds is
clean for a seed note containing a planted injection attempt, a test
asserting an over-length field is rejected, and a test attempting a
write/delete call with the new query-scoped key that asserts Pinecone
itself rejects it. Satisfies LLM01's Path B closure and LLM08's "blocked"
target. Landed before the direct-injection suite and citation-hardening
phases below, since both write tests that model the surface this phase
changes.

**Block 5 — direct-injection test suite.** Feeds Block 5's query-parsing
step adversarial inputs (jailbreak attempts, requests to reveal the system
prompt, attempts to steer parsed arguments) and asserts the parsed output
against the plausibility rule defined in the spec's LLM01 section
(checked against real values in the Neo4j graph, not an external
vocabulary), with anything that fails surfaced in tracing. Satisfies
LLM01's "direct injection: flagged" target — this is
also the piece that most literally satisfies the block's acceptance
criterion of showing prompt-injection attempts get blocked or flagged.

**Block 6 — citation hardening.** Adds the sanitization step where
citations are constructed (strips control sequences and instruction-like
patterns) and trims citation payloads using the sentence-level
keyword-containment rule defined in the spec's LLM02 section. Proves
itself two ways: a unit test on the sanitization function directly, and
an end-to-end test that plants an injection attempt in a seed patient
note and runs it through the real pipeline (Block 1 → Block 3 → Block 4 →
Block 6) to confirm it's neutralized by the time it reaches
`MultiAgentAnswer.citations` — plus a check that Block 4's eval harness
assertions on citation content still pass after trimming (the open
follow-up already flagged in the spec). Satisfies LLM01's "indirect
injection: blocked" and LLM02's "field-layer: blocked" targets together.

**Block 6 — state validation.** Adds schema/type validation at
`MultiAgentState` node boundaries so a malformed write gets caught and
logged instead of silently reaching reconciliation. Also wraps
`reconcile_node_safe`'s call to `_reconcile_error_answer` in its own
try/except, falling back to a fixed literal `MultiAgentAnswer` if that
helper itself fails, so nothing in the reconciliation error path can
crash `run_multi_agent` — found during review, folded into this same
phase since it's the same file and the same risk category. Proves
itself with two tests: one that deliberately writes a bad value at a node
boundary and confirms it's caught, and one that forces
`_reconcile_error_answer` to raise and confirms the system still returns
a valid answer instead of crashing. Satisfies LLM06's "state tampering:
flagged" and "reconciliation error-handling gap: blocked" targets.

**Block 6 — Cohort agent injection test and query-size visibility.** Adds
a dedicated Cypher-injection test for the Cohort agent's query, matching
Block 5's existing rigor (asserting on actual query structure and
parameters, not just "no crash"), plus logging how many rows the query
returns and how long it takes, alerting at the row-count and runtime
thresholds defined in the spec's LLM10 section (defaults pending
confirmation against the real patient population size). This is a new,
narrow signal about a single
query's size and runtime — separate from, and not a change to, Block 5's
existing LLM token/cost tracking. Satisfies LLM06's "Cypher injection:
blocked" target — this is the piece that most literally satisfies the
block's acceptance criterion around data-exfiltration attempts — and
LLM10's "exhaustive query: flagged" target.

**Block 6 — dependency pin bump.** Updates `requirements.txt` to
reference Block 5's new retry-hardened commit once that phase merges, so
the Clinical agent actually benefits from it. A version bump, not a
feature change — its own minimal PR rather than folded into another
phase.

**Block 7 — SECURITY.md.** Written last, after the phases above land,
because it documents what's actually true and tested, not what's planned.
Covers: the RBAC gap and the named Enterprise/Aura migration path (LLM02),
the cross-repo pinning policy (LLM03), and a short status line per risk
pointing at the test that proves its blocked/flagged claim.

## 4. Suggested order

Retry hardening (Block 5) first — it's porting an already-working pattern,
lowest risk, fastest. Chunk sanitization and key scoping (Block 4) comes
next, ahead of any phase that writes tests against the injection/input
surface, since it changes what those tests need to model — the
direct-injection suite and citation-hardening phases below both depend on
this ordering, per review feedback. The dependency pin bump (Block 6) can
land any time after retry hardening merges — it doesn't depend on Block 4.
After that, the remaining phases — citation hardening, state validation,
and the Cohort agent injection test with query-size visibility, all in
Block 6 — plus the direct-injection suite in Block 5, can happen in any
order relative to each other; they touch different code paths and don't
depend on one another. SECURITY.md is always last, after everything else
has merged.

## 5. What doesn't get its own phase

The RBAC migration recommendation and the supply-chain pinning policy are
both pure documentation — no code changes back them, so they're written
directly into SECURITY.md rather than getting a phase of their own.
Deciding whether Block 4's chunk sanitization happens at ingestion or at
prompt-build time is a small decision made at that phase's start, same as
the other in-phase decisions above.
