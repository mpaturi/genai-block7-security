# Block 7 — Security & Hardening: Threat Model Spec

## 1. Scope

System under test: the Block 6 multi-agent system (orchestrator, Block 5's
clinical agent reused unmodified, the new Cohort Enumeration Agent) and its
direct runtime dependencies — Block 5's tools, Block 4's RAG service
(FastAPI + Pinecone), Block 3's Neo4j graph. Block 1/2 batch pipelines are
out of scope except where they bear on corpus content integrity, noted
below as an adjacent risk.

Method: for each risk, state what's actually true in this codebase today
(with the specific mechanism), what already mitigates it (if anything), and
what Block 7 will build — distinguishing "theoretically in an OWASP
category" from "actually exploitable in this codebase today." Where a risk
can't be fully closed (e.g. by a limitation of Neo4j Community Edition),
the spec says so plainly instead of re-flagging it as if it were still
open-ended.

Five primary risks are mapped against OWASP Top 10 for LLM Applications
(2025). Two adjacent risks are named but treated as secondary, since they
belong more to Block 3/4's ingestion boundary than to Block 5/6's runtime.
Three further categories are deliberately absent rather than silently
skipped: LLM05 (Improper Output Handling) is not given its own section
because its concerns here are already covered under LLM01 and LLM06 below;
LLM07 (System Prompt Leakage) is folded into LLM01's direct-injection test
cases rather than treated as a separate surface; and LLM09 (Misinformation)
is not given its own section because the one failure mode Block 6's
reconciliation design directly addresses — the two agents disagreeing —
already gets a discrepancy flag and graceful degradation at the
architecture level. This is a partial mitigation, not a full one: it does
not cover a single agent stating a wrong number confidently when both
agents' underlying tool outputs agree, since there is no disagreement to
flag in that case. That residual gap is noted here rather than assumed
away, and is not addressed by Block 7.

## 2. Primary risks

### LLM01:2025 — Prompt Injection

Two surfaces exist here and must not be conflated.

Direct: the user's natural-language question ("of patients with [condition]
and [lab] [comparison] [value], how many are on [drug_a] vs [drug_b]?") is
parsed by Block 5's agent LLM into structured tool arguments before any
parameterization defense applies. This path is untested against injected
instructions today — e.g. "ignore prior instructions," attempts to steer
the model into calling tools with attacker-chosen values, or attempts to
get the model to reveal its system prompt. This is a live, user-controlled
surface, not a hypothetical one.

Indirect: raw patient note text (`chunk_text`) flows unsanitized into
`MultiAgentAnswer.citations`. Verified independently from both directions
against the current code (re-confirmed directly in response to review
feedback, not just carried over from an earlier claim). In Block 6
(`orchestrator.py`), all nine places `MultiAgentAnswer.answer` gets set
were traced: five are hardcoded strings for failure modes, one builds an
f-string purely from the Cohort agent's counts and the question's
structured fields (Role 2 has no note-text field to draw from), and the
remaining four copy `clinical_result.answer` verbatim from Block 5. In
Block 5 (`agent.py:51-83`, `_default_answer_fn` — the only place Block 5
calls an LLM to produce this text), the prompt is built from exactly four
inputs: the question's structured fields, a list of integer patient IDs,
and two integer drug counts — it never receives `rag_citations` or any
chunk/snippet content as an argument. So `chunk_text` does **not** reach
an LLM prompt anywhere in Block 5 or Block 6's current code — it only
ever reaches `MultiAgentAnswer.citations`, a separate field, never
`.answer`. This is dormant today, not live. It becomes live the moment
something — a future UI, or this project's own Block 8 capstone — reads
citations back into a model or renders them raw.

**Decision:** sanitize proactively in Block 7, not test-and-defer. A
neutralization step is added where citations are constructed (before they
enter `MultiAgentAnswer`), so Block 8 inherits an already-safe field
instead of a live latent risk.

**Target:** direct injection attempts are *flagged* — tests assert parsed
tool arguments stay within expected domain/type/range (e.g. a parsed
"condition" is a plausible clinical term, not an instruction fragment); any
case where a jailbreak still produces a validly-typed argument must still
surface in tracing, since type-validity alone doesn't prove the argument is
legitimate. Indirect injection is *blocked* — the sanitization step strips
or neutralizes control sequences and instruction-like patterns before they
ever leave the field, proven two ways: a unit-level regression test on
the sanitization function itself, and an end-to-end test that plants an
injection attempt in a seed patient note and runs it through the real
pipeline (Block 1's note generation, Block 3's storage, Block 4's
retrieval, Block 6's citation construction) to confirm it's neutralized by
the time it reaches `MultiAgentAnswer.citations`. The end-to-end version
is the stronger proof — it tests the real system end to end, not an
isolated function, and is only possible because the corpus is fully
controlled.

**Plausibility rule:** after parsing, check the condition, lab name, and
drug names against the actual set of values present in the Neo4j graph
(queried once and cached, not re-queried per request). A parsed value
that doesn't match anything in that real list gets flagged as suspicious.
This checks against your own data, not an external medical vocabulary —
simpler, and more accurate for this specific dataset.

### LLM02:2025 — Sensitive Information Disclosure

Two layers.

Data-layer access control: Neo4j Community Edition has zero RBAC support
(confirmed directly — `SHOW ROLES` fails outright). Application-level
Cypher parameterization is the *only* enforced defense; anyone holding
driver credentials has unrestricted read/write regardless of what the
app's own queries look like. This is a real, currently-true single point
of failure, not a hypothetical one.

Field-layer over-exposure: citations currently return `chunk_text`
verbatim, which may carry more patient note content than an answer
requires — a data-minimization gap independent of prompt injection.

**Decision:** accept the RBAC gap explicitly as a documented residual risk
— it cannot be structurally fixed on Community Edition — **and** name a
concrete migration path (Neo4j Enterprise or Aura with a scoped read-only
role) as explicit future work, rather than re-flagging the same gap with
no path forward.

**Target:** the RBAC gap is *flagged/documented*, not blocked, in this
block. Field-layer over-exposure is *blocked* — citation payloads are
trimmed to what the answer actually needs, built alongside the
sanitization work in LLM01 above (same code path).

**Trimming rule:** split `chunk_text` into sentences; keep only the
sentences that contain at least one of the parsed query terms (the
condition, lab name, or drug names extracted from the question). If no
sentence contains a match, keep the first sentence only, as a safe
default rather than dropping the citation entirely. Deliberately simple —
a keyword-containment filter, no additional model calls, no fuzzy
re-ranking — consistent with keeping the code readable.

### LLM03:2025 — Supply Chain

Block 6's `requirements.txt` pins its dependency on Block 5 to a specific
git branch reference rather than a versioned package release — a
cross-repo dependency pattern inherent to this project's one-repo-per-block
structure. Any time that pin is updated to point at a different commit or
branch, Block 6 silently inherits whatever code is at that reference, with
none of the review gate a published library version would normally have.

**Target:** *flagged*, not blocked — this isn't something a runtime test
can catch, it's a practice to hold to. Block 7 states the policy plainly in
SECURITY.md: cross-repo pins must always reference reviewed, merged
commits or tags on `main`, never an active working branch, and any change
to a pin goes through the same phase-branch/PR process as any other code
change.

### LLM06:2025 — Excessive Agency

Three items. Two are Block 6's own `spec.md` already naming and
explicitly deferring this to Block 7 — not new scope. The third was
found during this spec's own review.

Orchestrator state tampering: `MultiAgentState` is a mutable `TypedDict`
passed between LangGraph nodes. If a compromised or simply buggy node
wrote an unexpected value into it, the reconciliation logic downstream
would trust it without validation.

Cypher injection on the new tool: the app-level parameterization defense
is dedicated-tested for Block 5's tools. It needs confirming that the
Cohort Enumeration Agent's query (new in Block 6) has its own dedicated
injection test — not just inherited assurance because it resembles Block
5's pattern.

Reconciliation error-handling gap (found during review, verified directly
against the code): `reconcile_node_safe` wraps `reconcile_node` in a
try/except that correctly prevents the main failure case from crashing
`run_multi_agent`. But that except block calls `_reconcile_error_answer`
unguarded — if that helper itself raises (e.g. a validation error while
constructing the fallback answer), the exception escapes uncaught, all
the way out of `run_multi_agent`. Low-probability today, since everything
currently fed into that helper is hardcoded or already-validated, but
structurally real, and nothing currently defends against it.

**Target:** state tampering is *flagged* — schema/type validation is added
at node boundaries in `MultiAgentState`, so a malformed write is caught and
logged rather than silently propagating into reconciliation. Cypher
injection on the new tool is *blocked* — a dedicated test, matching Block
5's existing rigor (asserting on actual query structure and parameter
dict, not just "no crash" or "no error"). The reconciliation error-handling
gap is *blocked* — the call to `_reconcile_error_answer` gets wrapped in
its own try/except, falling back to a fixed literal `MultiAgentAnswer`
with no computed fields if the helper itself fails, so nothing in this
path can crash `run_multi_agent`. Proven by a test that forces the inner
helper to raise and confirms the system still returns a valid answer
instead of propagating the exception.

### LLM10:2025 — Unbounded Consumption

Two amplifiers.

Block 5's zero-delay retries (known, already-disclosed gap): any transient
failure — LLM rate limit, Neo4j timeout — retries immediately and
repeatedly. Under real failure conditions this is a cost and availability
amplifier, not just a code-quality nit.

The Cohort Enumeration Agent's query is intentionally exhaustive by design
(no `top_k` ceiling — that's the point, it's what closes Block 5's
undercounting gap). Correct for correctness, but it also means a
pathological or adversarial query (a condition matching a large fraction
of the graph) has no cap on result size or compute cost.

**Decision:** retry/backoff hardening for Block 5 is in scope for Block 7,
closing the inconsistency against the standard Block 6 Phase 8 already
established.

**Target:** the retry gap is *blocked* — Block 5's retry logic is brought
up to Block 6 Phase 8's standard (exponential backoff, exception
classification), verified by tests that simulate transient failures and
assert retry timing/count. The exhaustive-query cost risk is *flagged*, not
capped — a hard limit would reintroduce the undercounting problem this
agent exists to solve, so instead result size/cost is logged with a
configurable soft-alert threshold, making an anomalously large result set
visible in tracing rather than silent.

**Retry policy (default, pending confirmation against Block 6's actual
Phase 8 numbers):** up to 3 retries (4 attempts total), exponential
backoff starting at 1 second and doubling each attempt (1s, 2s, 4s), then
give up and surface a clear error. Retryable: rate-limit responses,
timeouts, connection errors. Not retryable, fail immediately: validation
errors, authentication errors, malformed requests. These numbers should
match Block 6's Phase 8 exactly for consistency — confirm against that
code when implementing, and update here first if they differ.

**Soft-alert threshold (default, pending confirmation against the real
patient population size):** flag a Cohort agent query as unusually large
if it returns more than 500 patients, or more than 25% of the total
patient population, whichever is smaller — chosen as well above the
largest cohort seen in eval so far (99 patients) to avoid false alarms on
normal broad queries. Also flag if a single query takes longer than 5
seconds to execute. Both numbers are starting points, not fixed facts —
tune them once the real dataset size is confirmed.

## 3. Adjacent risks (named, not primary)

**LLM04:2025 — Data and Model Poisoning.** If Block 1/2's ingestion doesn't
validate patient note *content* (only schema/type today), an adversarial
note could be embedded and later retrieved as a citation — the source of
the LLM01 indirect-injection surface above. This is a Block 3/4 ingestion
concern, not a Block 5/6 runtime one; noted here so it isn't lost, not
addressed in Block 7.

**LLM08:2025 — Vector and Embedding Weaknesses.** Checked directly:
Pinecone does support read-only API keys (the `DataPlaneViewer` role —
query, fetch, list, and stats only, no write or delete), unlike Neo4j
Community Edition, which has no such option at all. Block 4's actual key
(named `default`) was checked in the Pinecone console and confirmed to
carry the `All` role — full read and write access to the entire project,
used by both the query path (`retrieve.py`, `api.py`) and the write path
(`create_index.py`, `ingest.py`, `verify.py`). Also verified: no other
repo holds or calls this key directly — Block 5 and Block 6 reach
Pinecone exclusively through Block 4's own API, so this gap is fully
contained to Block 4. This is a real, cheaply fixable gap, not a
structural limitation like the Neo4j one — fixing it would mean splitting
the query and write paths onto separate keys, a small code change, not a
migration. Deliberately not addressed in Block 7: fixing it requires
touching Block 4's code, which is out of scope for this block.

## 4. Explicitly out of scope for Block 7

- Content-level validation at ingestion (LLM04, above) — corpus integrity
  is assumed upstream of Block 6.
- Pinecone API key scoping (LLM08, above) — confirmed to be a real,
  fixable gap, fully contained to Block 4, but fixing it requires a
  Block 4 code change that's out of scope for this block.
- Full DB-level RBAC implementation — documented and a migration path
  named, not implemented; Community Edition cannot do it, and an
  Enterprise/Aura migration is a decision beyond this curriculum block.
- General non-security refactors unrelated to the risks above.

## 5. What "blocked" vs "flagged" means for Block 7's test suite

**Blocked:** an automated test proves the malicious input cannot achieve
its effect at the structural level. Example: a Cypher-injection-shaped
payload passed as a `condition` parameter is asserted to execute only as a
literal parameter value — the test checks the actual parameterized query
structure and parameter dict sent to the driver, not just "the app didn't
crash" or "zero rows came back" (either could pass by coincidence).

**Flagged:** the request is allowed to complete, but produces a visible,
structured signal that a human or downstream system can act on — a field
on the response (e.g. a validation warning attached to `MultiAgentState`),
a LangSmith trace annotation, or a log line tied to a specific check.
Silent completion with no signal does not count as "flagged," even if the
underlying behavior was technically safe.

Honest limit on "flagged": nothing in Block 7's scope proactively
surfaces these signals to anyone — there's no dashboard or alerting yet.
That's Block 8's observability work. Until then, "flagged" means
discoverable on inspection, not actively monitored.

## 6. Open follow-ups

- Confirm citation field-minimization (LLM02, field layer) doesn't break
  Block 4's existing eval harness assertions on citation content — those
  tests may currently assert on full `chunk_text`.
- Confirm the retry policy numbers (LLM10) against Block 6's actual
  Phase 8 code — the spec's 3-retries/1s-2s-4s defaults need matching to
  the real implementation, not just assumed.
- Confirm the soft-alert threshold (LLM10, 500 patients or 25% of
  population) against the real total patient count in the graph — the
  default was chosen without knowing that number.

## 7. Next step

This spec defines *what* Block 7 addresses and *how blocked/flagged are
judged*. `docs/plan.md` (next, not built yet) breaks each target above into
phases, branches, and the specific test files each phase adds — per house
rules, spec first, then plan, then tasks, then code.
