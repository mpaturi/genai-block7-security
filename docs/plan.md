# Block 7 — Security & Hardening: Implementation Plan

This plan turns `docs/spec.md`'s risks and decisions into phases. It
deliberately does not pin exact file paths or line-level detail — those
get confirmed against the real, merged code at the start of each phase,
not guessed at here. Nothing in this document has been committed anywhere;
it's a draft for review alongside the spec.

## 1. Before any phase starts

Once Block 5's PRs #12/#13 and Block 6's PR #9 are merged, the first step
is not writing code — it's a five-minute check that the spec's code-level
claims (how citations get built, the shape of `MultiAgentState`, the
Cohort agent's query, what Block 5's answer-writing call receives) still
match `main` in both repos. If reviewer feedback changed any of those
during review, the spec gets a small factual correction here, before any
implementation branch opens.

## 2. Which repo each fix lands in

**`genai-block7-security`** (no runtime code): `docs/spec.md`,
`docs/plan.md` (this document), and `SECURITY.md`. This repo documents
and tests the threat model; it doesn't own any of the application code
being hardened.

**`genai-block6-multiagent`**: citation sanitization and field-minimization
(LLM01 indirect, LLM02 field layer — same code path, one phase), state
validation at `MultiAgentState` boundaries (LLM06 tampering), the Cohort
agent's dedicated injection test plus exhaustive-query cost logging
(LLM06 injection, LLM10 exhaustive query — same code area, one phase).
These are Block 6's own objects; the fixes live where the objects live.

**`genai-block5-agent`**: retry/backoff hardening (LLM10 retry gap) and
the direct-injection test suite (LLM01 direct). Both target Block 5's own
agent code — the retry logic and the query-parsing step are Block 5's,
not Block 6's, even though Block 6 depends on them. Tests live with the
code they test, same as Block 5 and Block 6's existing parameterization
tests already do.

## 3. Phases

Each phase is one branch, one focused PR, per house rules. Phase numbers
aren't assigned here — each repo continues its own existing sequence
(e.g. Block 6's next phase follows whatever number PR #9 lands as; Block
5's next phase follows PR #13). Assign the real number when the branch
is actually cut, not now.

**Block 5 — retry/backoff hardening.** Ports Block 6's Phase 8 pattern
(exponential backoff, exception classification) into Block 5's tool-calling
code. Proves itself with tests that simulate a transient failure and
assert the wait grows between attempts and the system gives up instead of
retrying forever. Satisfies LLM10's "retry gap: blocked" target.

**Block 5 — direct-injection test suite.** Feeds Block 5's query-parsing
step adversarial inputs (jailbreak attempts, requests to reveal the system
prompt, attempts to steer parsed arguments) and asserts the parsed output
stays within expected type/domain/range, with anything suspicious surfaced
in tracing. Satisfies LLM01's "direct injection: flagged" target — this is
also the piece that most literally satisfies the block's acceptance
criterion of showing prompt-injection attempts get blocked or flagged.

**Block 6 — citation hardening.** Adds the sanitization step where
citations are constructed (strips control sequences and instruction-like
patterns) and trims citation payloads to only what an answer needs.
Proves itself with a regression test checking this structurally, not just
against today's known-safe call graph — plus a check that Block 4's eval
harness assertions on citation content still pass after trimming (the
open follow-up already flagged in the spec). Satisfies LLM01's "indirect
injection: blocked" and LLM02's "field-layer: blocked" targets together.

**Block 6 — state validation.** Adds schema/type validation at
`MultiAgentState` node boundaries so a malformed write gets caught and
logged instead of silently reaching reconciliation. Proves itself with a
test that deliberately writes a bad value at a node boundary and confirms
it's caught, not trusted. Satisfies LLM06's "state tampering: flagged"
target.

**Block 6 — Cohort agent injection test and cost visibility.** Adds a
dedicated Cypher-injection test for the Cohort agent's query, matching
Block 5's existing rigor (asserting on actual query structure and
parameters, not just "no crash"), plus result-size/cost logging with a
configurable soft-alert threshold. Satisfies LLM06's "Cypher injection:
blocked" target — this is the piece that most literally satisfies the
block's acceptance criterion around data-exfiltration attempts — and
LLM10's "exhaustive query: flagged" target.

**Block 7 — SECURITY.md.** Written last, after the phases above land,
because it documents what's actually true and tested, not what's planned.
Covers: the RBAC gap and the named Enterprise/Aura migration path (LLM02),
the cross-repo pinning policy (LLM03), and a short status line per risk
pointing at the test that proves its blocked/flagged claim.

## 4. Suggested order

Retry hardening first — it's porting an already-working pattern, lowest
risk, fastest, and unblocks a small follow-up: once it merges, Block 6's
`requirements.txt` pin should bump to reference the new Block 5 commit,
so the Clinical agent actually benefits from it (small, separate commit,
following the pinning policy the spec already states). After that, the
two Block 6 phases and the direct-injection suite can happen in any order
— they touch different code paths and don't depend on each other. SECURITY.md
is always last.

## 5. What doesn't get its own phase

The RBAC migration recommendation and the supply-chain pinning policy are
both pure documentation — no code changes back them, so they're written
directly into SECURITY.md rather than getting a phase of their own.