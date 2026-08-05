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

Six primary risks are mapped against OWASP Top 10 for LLM Applications
(2025). One further risk is named but treated as adjacent, since it
belongs more to Block 1/2's ingestion boundary than to Block 5/6's
runtime. Three further categories are deliberately absent rather than
silently skipped: LLM05 (Improper Output Handling) is not given its own section
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

Direct: corrected during this review — there is no natural-language
question anywhere in this system, and no LLM anywhere parses free text
into structured tool arguments. `QuestionInput`'s `condition`, `lab`,
`drug_a`, and `drug_b` fields are free-text strings a caller supplies
already structured. `block5_agent/schemas.py`'s `build_rag_query()` and
`assemble_question_text()` f-string these values directly into the
Pinecone search text and into the `"Question: ..."` line inside
`_default_answer_fn`'s prompt to Claude (`agent.py`). `QuestionInput`
(`block5_agent/schemas.py`) now bounds these fields —
`condition` at `max_length=200`, `lab`/`drug_a`/`drug_b` at
`max_length=100`, and `value` at `ge=0, le=10_000` (also rejecting
`inf`/`-inf`/`nan`, which otherwise crash the response renderer entirely —
see LLM10 below). These bounds close two narrower risks — oversized
payloads and non-finite numeric input — but are not content-level
injection filtering: a value well within 200 characters and still
entirely instruction-like text passes through unchanged. So whatever a
caller puts in `condition`/`lab`/`drug_a`/`drug_b`, within these bounds,
still reaches an LLM prompt verbatim, via direct string interpolation,
not model-mediated parsing. This path remains untested against injected
content today — e.g. instruction-like text placed in `condition`, or
attempts to make the assembled question text read as a new instruction
rather than a clinical term. This is a live, caller-controlled surface,
not a hypothetical one — the earlier framing of this as an LLM-parsing
step described a flow that doesn't exist anywhere in Blocks 5, 6, or 8.

Indirect: raw patient note text (`chunk_text`) has two separate downstream
paths, and they must not be conflated.

Path A — citations: `chunk_text` flows into `MultiAgentAnswer.citations`
unsanitized. Verified independently against the current code. In Block 6
(`orchestrator.py`), all nine places `MultiAgentAnswer.answer` gets set
were traced: four are hardcoded strings for failure modes, one builds an
f-string purely from the Cohort agent's counts and the question's
structured fields (Role 2 has no note-text field to draw from), and the
remaining four copy `clinical_result.answer` verbatim from Block 5. In
Block 5 (`agent.py:51-83`, `_default_answer_fn` — the only place Block 5
calls an LLM to produce this text), the prompt is built from exactly four
inputs: the question's structured fields, a list of integer patient IDs,
and two integer drug counts — it never receives `rag_citations` or any
chunk/snippet content as an argument. So along this path, `chunk_text`
does **not** reach an LLM prompt anywhere in Block 5 or Block 6's current
code — it only ever reaches `MultiAgentAnswer.citations`, a field
distinct from `.answer`.

Path B — Block 4's own generation call: `chunk_text` reaches a live LLM
prompt today, on every answerable question, not a hypothetical or future
one. Block 5's `search_patients` calls Block 4's `POST /query` on every
question. Block 4's `generate_answer` (`generate.py:57-61`) builds its
user message by interpolating `chunk['chunk_text']` directly into the
prompt, and this is called unconditionally at `api.py:116` whenever any
retrieved chunk clears the relevance threshold. This path was missed in
an earlier draft of this section, which assessed the overall indirect-
injection risk as dormant based only on tracing Blocks 5 and 6 — one repo
short of where this exposure actually sits.

Impact of Path B is genuinely limited: Block 5 only reads back
`person_id`/`chunk_id`/`score`/`chunk_text` as structured `sources` from
Block 4's response — it never consumes Block 4's own generated `answer`
text — and `MultiAgentAnswer.answer` is written exclusively by Block 5's
own `_default_answer_fn` (Path A above). So an injected instruction in a
note has little leverage over what the end user actually reads. But Path
B is live today, on every query, not dormant — the risk here is real even
though its blast radius is small.

Also found during this review: Block 4's `QueryRequest` (`api.py`) has no
`max_length` on any of its four free-text fields (`question`,
`condition`, `drug`, `lab`). Of these, only `question` reaches an LLM
prompt directly — it's interpolated into `generate_answer`'s prompt
alongside `chunk_text`, so it's the same surface as Path B above.
`condition`, `drug`, and `lab` are used exclusively as Pinecone metadata
filter values in `build_metadata_filter` (`retrieve.py`) and never reach
an LLM prompt — capping them is a narrower input-hygiene measure (an
oversized filter value), not a prompt-injection fix. All four still get
a `max_length`, for two different reasons, not one.

**Decision:** sanitize proactively, not test-and-defer, and close both
paths, not just the one originally found. The primary fix moves to where
the live exposure actually is: neutralize `chunk_text` either at
ingestion into Pinecone or inside Block 4's `generate_answer` before its
prompt is built — this brings `genai-block4-rag-eval` into Block 7's
scope for this one fix, alongside a `max_length` cap on the four fields
above. The Block 6 citation-sanitization step (Path A) is kept as a
second layer protecting the rendering path, even though that path doesn't
reach an LLM prompt today.

**Target:** direct injection attempts are *flagged* — tests assert the
caller-supplied `condition`/`lab`/`drug_a`/`drug_b` values stay within
expected domain (e.g. `condition` is a plausible clinical term, not an
instruction fragment); a value that's validly-typed and within length
limits but still implausible as a real clinical term must still surface
in tracing, since type-validity and length alone don't prove a value is
legitimate. Indirect injection is *blocked* on both paths: Block 4's
`generate_answer` never receives unsanitized `chunk_text` (proven by a
test asserting the constructed prompt is clean for a seed note containing
an injection attempt), and `MultiAgentAnswer.citations` is separately
sanitized and trimmed, proven two ways — a unit-level regression test on
the sanitization function itself, and an end-to-end test that plants an
injection attempt in a seed patient note and runs it through the real
pipeline (Block 1's note generation, Block 3's storage, Block 4's
retrieval, Block 6's citation construction) to confirm it's neutralized by
the time it reaches `MultiAgentAnswer.citations`. The end-to-end version
is the stronger proof — it tests the real system end to end, not an
isolated function, and is only possible because the corpus is fully
controlled. `MultiAgentAnswer.answer`/`.caveat` get the same structural
sanitization Block 6 applies to citations, applied at every point they're
assigned into a `MultiAgentAnswer` — verified by enumerating all nine
`.answer`-construction sites in `orchestrator.py`: four hardcoded failure
strings (safe, no caller or model input), four that copy already-sanitized
text from Block 5's `ClinicalAnswer.answer`/`.caveat`, and
`_cohort_only_degraded_answer`, which f-strings the question's own
caller-supplied fields (`condition`/`lab`/`comparison`/`value`/`drug_a`/
`drug_b`) into the degraded-mode message with no LLM involved at all.
This last site was found unsanitized during this review — it fell
outside the original fix's LLM-steering framing, since no model call
happens on this path — and is now wrapped in the same
`sanitize_citation_text` call as every other site. This free text is
exactly as unverified as a citation snippet; for the four Block-5-sourced
sites it's in fact the more direct surface for a successfully-steered
injection to reach a caller, since it's Claude's own generated words, not
a retrieved excerpt. The cohort-only-degraded site is a different case —
caller input echoed straight back with no model in the loop — but gets
the same defense regardless, since it's the same class of unverified free
text reaching the same caller-facing field.

**Assumption this relies on:** Block 5's `run_agent` is never called
directly by anything that hands its raw, unsanitized
`ClinicalAnswer.answer`/`.caveat` straight to a caller — confirmed against
the real code, not assumed: `genai-block8-capstone/app/api.py` only
imports `QuestionInput` from `block5_agent`, never `run_agent` itself, and
reaches Block 5 exclusively through `run_multi_agent_async`
(`scripts/orchestrator.py`), the one place this sanitization runs. No
other file in Block 8 calls `run_agent` at all (checked directly). If a
future caller ever invokes Block 5's agent standalone, bypassing Block 6,
this sanitization would not apply — that caller would need its own layer,
not something to retrofit into Block 5 preemptively today.

**Plausibility rule:** check the caller-supplied condition, lab name, and
drug names against the actual set of values present in the Neo4j graph
(queried once and cached, not re-queried per request). A value that
doesn't match anything in that real list gets flagged as suspicious. This
checks against the system's own data, not an external medical vocabulary
— simpler, and more accurate for this specific dataset.

### LLM02:2025 — Sensitive Information Disclosure

Three layers.

Data-layer access control: Neo4j Community Edition has zero RBAC support
(confirmed directly — `SHOW ROLES` fails outright). Application-level
Cypher parameterization is the *only* enforced defense; anyone holding
driver credentials has unrestricted read/write regardless of what the
app's own queries look like. This is a real, currently-true single point
of failure, not a hypothetical one.

Field-layer over-exposure: citations currently return `chunk_text`
verbatim, which may carry more patient note content than an answer
requires — a data-minimization gap independent of prompt injection.

Network/API-layer access control: neither Block 4's `POST /query` nor
Block 8's own API has any authentication or access control — confirmed
directly against `scripts/api.py` and `app/api.py`, neither has an auth
dependency, API key check, or middleware of any kind. Block 4's own
`docs/spec.md` already lists this under Scope as a conscious exclusion;
Block 8's does not mention it at all. Today this is contained, not
exploitable over a network: `genai-block8-capstone/docker-compose.yml`
binds every service — Neo4j, the RAG service, the app — to `127.0.0.1`
only, never `0.0.0.0`, and Block 4's own documented run command
(`uvicorn scripts.api:app --reload`) defaults to the same loopback-only
bind. The gap is real at the code level, but the deployment as actually
configured is the only thing standing between it and being reachable —
redeploying with different port bindings (a real cloud host, a container
orchestrator that exposes ports by default) would expose it immediately,
with no code-level change required to trigger that.

**Decision:** accept the RBAC gap explicitly as a documented residual risk
— it cannot be structurally fixed on Community Edition — **and** name a
concrete migration path (Neo4j Enterprise or Aura with a scoped read-only
role) as explicit future work, rather than re-flagging the same gap with
no path forward. Accept the network/API-layer gap the same way — name a
concrete path (an API key dependency on both FastAPI apps via `Depends()`,
or auth terminated at a reverse proxy in front of both services) as
explicit future work, rather than leaving it undocumented, which is the
actual gap being closed here — the missing auth itself is unchanged by
this phase, only its documentation is.

**Target:** the RBAC gap and the network/API-layer gap are both
*flagged/documented*, not blocked, in this block — neither is something a
runtime test can close, and the network-layer gap specifically is
mitigated by deployment configuration, not application code, so no
application-level test could prove it either way. Field-layer
over-exposure is *blocked* — citation payloads are trimmed to what the
answer actually needs, built alongside the sanitization work in LLM01
above (same code path).

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

### LLM08:2025 — Vector and Embedding Weaknesses

Checked directly: Pinecone does support read-only API keys (the
`DataPlaneViewer` role — query, fetch, list, and stats only, no write or
delete), unlike Neo4j Community Edition, which has no such option at all.
Block 4's actual key (named `default`) was checked in the Pinecone
console and confirmed to carry the `All` role — full read and write
access to the entire project, used by both the query path (`retrieve.py`,
`api.py`) and the write path (`create_index.py`, `ingest.py`,
`verify.py`). Also verified: no other repo holds or calls this key
directly — Block 5 and Block 6 reach Pinecone exclusively through Block
4's own API, so this gap is fully contained to Block 4.

**Decision:** fix it directly rather than only documenting it. This is a
real, cheaply fixable gap, not a structural limitation like the Neo4j
one, and Block 4 is already in scope for this revision's LLM01 fix (see
above), so the same phase carries this change too. Split the query and
write paths onto separate keys: a `DataPlaneViewer`-scoped key for
`retrieve.py`/`api.py`, and the existing full-access key retained only
for `create_index.py`/`ingest.py`/`verify.py`.

**Target:** *blocked* — the query path can no longer write or delete
regardless of what code runs against it, proven by a test that attempts a
write/delete call using the query-path key and asserts Pinecone itself
rejects it, not just application logic.

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

An unbounded Cypher call (found during Block 8's PR review, verified
directly against the code): `scripts/vocabulary_check.py`'s
`_fetch_known_vocabulary()` — called via `reconcile_node`'s
`get_known_vocabulary()` in `orchestrator.py` — issues its two queries
with a plain `session.run()`, unlike every other query in this repo
(`cohort_tool.py`'s `query_full_cohort`/`count_drugs_exhaustive`), which
wrap their calls in `Query(..., timeout=...)`. Since `reconcile_node` runs
only after both branches have already returned (past Block 6's own 150s
branch-level timeout), a wedged Neo4j at exactly this point hangs
indefinitely with no outer bound in this repo. Block 8 added its own
180s caller-side timeout as a mitigation, but the actual gap is here.

A related but distinct availability gap, found and fixed during this
review: `QuestionInput.value`/`QueryRequest.value` accepted any float,
including `inf`/`-inf`/`nan`. Beyond the missing domain bound itself,
rejecting an out-of-range value the normal way exposed a second,
independent bug — FastAPI/Starlette's default `RequestValidationError`
handler embeds the raw rejected value back into the 422 response body,
and `JSONResponse.render()` calls `json.dumps(..., allow_nan=False)`,
which raises on a non-finite float and turns a should-be-422 into an
unhandled 500. Fixed in Block 4 and Block 8 (Block 5 and Block 6 have no
HTTP layer of their own for this to crash) with a `ge=0, le=10_000` bound
on `value` plus a custom `RequestValidationError` handler that stringifies
the error instead of re-encoding the raw rejected value. Noted here
rather than under LLM01 because the actual failure mode is availability —
an unhandled crash — not content injection.

**Decision:** retry/backoff hardening for Block 5 is in scope for Block 7,
closing the inconsistency against the standard Block 6 Phase 8 already
established. The unbounded vocabulary-check query is also in scope —
same fix pattern (`Query(..., timeout=...)`) already used everywhere else
in `cohort_tool.py`, just not yet applied to this one call site.

**Target:** the retry gap is *blocked* — Block 5's retry logic is brought
up to Block 6 Phase 8's standard (exponential backoff, exception
classification), verified by tests that simulate transient failures and
assert retry timing/count. The unbounded vocabulary-check query is also
*blocked* — a `timeout=` is added to both `session.run()` calls in
`_fetch_known_vocabulary()`, matching `cohort_tool.py`'s existing
`GRAPH_QUERY_TIMEOUT` pattern/value, verified against a genuinely slow
query surfacing as a timeout error rather than hanging. The
exhaustive-query cost risk is *flagged*, not capped — a hard limit would
reintroduce the undercounting problem this agent exists to solve, so
instead result size/cost is logged with a configurable soft-alert
threshold, making an anomalously large result set visible in tracing
rather than silent.

**Retry policy (confirmed against Block 6's actual Phase 8 code —
`cohort_agent.py`'s `_MAX_TOOL_RETRIES`/`_RETRY_BACKOFF_SECONDS`):** up to
2 retries (3 attempts total), linear backoff of `0.5 * (attempt + 1)`
seconds (0.5s, then 1.0s), then give up and surface a clear error. Block
5's answer-writing step keeps its own smaller, already-existing budget of
1 retry (2 attempts total) rather than adopting the 3-attempt tool
budget, since a retry there is a second, slower language-model call — it
uses the same backoff formula, just with one gap (0.5s) instead of two.
Retryable: rate-limit responses, timeouts, connection errors. Not
retryable, fail immediately: validation errors, authentication errors,
malformed requests.

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

## 4. Explicitly out of scope for Block 7

- Content-level validation at ingestion (LLM04, above) — corpus integrity
  is assumed upstream of Block 6.
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

- ~~Confirm citation field-minimization (LLM02, field layer) doesn't
  break Block 4's existing eval harness assertions on citation
  content~~ — resolved in Block 6's citation-hardening phase: the full
  eval harness was re-run after trimming landed (9/9 recall, no
  regression, no broken citation assertions).
- Confirm the soft-alert threshold (LLM10, 500 patients or 25% of
  population) against the real total patient count in the graph — still
  open. Block 6's Cohort agent phase implemented the threshold with an
  explicit placeholder (`_ASSUMED_TOTAL_PATIENT_POPULATION = 10_000`,
  documented as a default to tune, not a confirmed number — the largest
  cohort seen in eval so far is 99 patients).
- ~~Create the new `DataPlaneViewer`-scoped Pinecone key (LLM08) in the
  Pinecone console~~ — resolved: the scoped key has been created and
  `tests/test_pinecone_key_scope.py` has been run against it directly
  (not just present in the suite — it skips itself without live
  credentials), confirming LLM08's "blocked" target for real.

## 7. Next step

This spec defines *what* Block 7 addresses and *how blocked/flagged are
judged*. `docs/plan.md` (next, not built yet) breaks each target above into
phases, branches, and the specific test files each phase adds — per house
rules, spec first, then plan, then tasks, then code.
