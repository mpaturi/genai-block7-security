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
`MultiAgentAnswer.citations`. Already traced precisely: it does **not**
reach an LLM prompt anywhere in Block 5 or Block 6's current code — Block
5's answer-writing call only receives patient IDs and drug counts; Block
6's reconciliation is pure Python string templating with no LLM call at
all. So this is dormant today, not live. It becomes live the moment
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
ever leave the field, proven by a regression test that checks this
structurally (not just against today's known-safe call graph).

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

Two items Block 6's own `spec.md` already named and explicitly deferred to
Block 7 — not new scope.

Orchestrator state tampering: `MultiAgentState` is a mutable `TypedDict`
passed between LangGraph nodes. If a compromised or simply buggy node
wrote an unexpected value into it, the reconciliation logic downstream
would trust it without validation.

Cypher injection on the new tool: the app-level parameterization defense
is dedicated-tested for Block 5's tools. It needs confirming that the
Cohort Enumeration Agent's query (new in Block 6) has its own dedicated
injection test — not just inherited assurance because it resembles Block
5's pattern.

**Target:** state tampering is *flagged* — schema/type validation is added
at node boundaries in `MultiAgentState`, so a malformed write is caught and
logged rather than silently propagating into reconciliation. Cypher
injection on the new tool is *blocked* — a dedicated test, matching Block
5's existing rigor (asserting on actual query structure and parameter
dict, not just "no crash" or "no error").

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

## 3. Adjacent risks (named, not primary)

**LLM04:2025 — Data and Model Poisoning.** If Block 1/2's ingestion doesn't
validate patient note *content* (only schema/type today), an adversarial
note could be embedded and later retrieved as a citation — the source of
the LLM01 indirect-injection surface above. This is a Block 3/4 ingestion
concern, not a Block 5/6 runtime one; noted here so it isn't lost, not
addressed in Block 7.

**LLM08:2025 — Vector and Embedding Weaknesses.** Pinecone access control
and embedding-inversion risk belong to Block 4's infrastructure, outside
Block 6's runtime. Noted for completeness, not in scope here.

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

## 6. Open follow-ups

- Confirm citation field-minimization (LLM02, field layer) doesn't break
  Block 4's existing eval harness assertions on citation content — those
  tests may currently assert on full `chunk_text`.

## 7. Next step

This spec defines *what* Block 7 addresses and *how blocked/flagged are
judged*. `docs/plan.md` (next, not built yet) breaks each target above into
phases, branches, and the specific test files each phase adds — per house
rules, spec first, then plan, then tasks, then code.
