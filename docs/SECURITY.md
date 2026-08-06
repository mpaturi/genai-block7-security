# Security Posture

Written last, after every phase below it references has actually merged —
not a plan, a record of what's actually true and tested in this codebase
today. See `docs/spec.md` for the full threat model and reasoning behind
each decision; this file states outcomes and points at the tests that
prove them.

## 1. Data-layer access control (Neo4j RBAC)

Neo4j Community Edition has no RBAC support (`SHOW ROLES` fails outright,
confirmed directly). Application-level Cypher parameterization is the
*only* enforced defense — anyone holding driver credentials has
unrestricted read/write regardless of what the application's own queries
look like. This is a structural limitation of Community Edition, not
something Block 7 can fix in code, and it's accepted here explicitly as a
documented residual risk rather than re-flagged with no path forward.

**Migration path:** Neo4j Enterprise Edition or Neo4j Aura, either of
which supports role-based access control. The concrete shape: a
scoped, read-only role for every application credential currently in use
(`genai-block5-agent`'s and `genai-block6-multiagent`'s driver
credentials), removing standing write access from any credential that
only ever runs `MATCH`/`RETURN` queries today. Not scheduled — this is
named as explicit future work, contingent on a Neo4j Enterprise or Aura
migration decision outside Block 7's scope.

## 2. Network / API-layer access control

Neither Block 4's `POST /query` nor Block 8's own API has authentication,
an API key check, or any middleware of any kind (confirmed directly
against `scripts/api.py` and `app/api.py`). Today this is contained, not
exploitable over a network, purely as a matter of deployment
configuration: `genai-block8-capstone/docker-compose.yml` binds every
service — Neo4j, the RAG service, the app — to `127.0.0.1` only, and
Block 4's documented run command (`uvicorn scripts.api:app --reload`)
defaults to the same loopback-only bind. The gap is real at the code
level; only the current deployment configuration stands between it and
being reachable. Redeploying either service with different port bindings
would expose it immediately, with no code change required to trigger
that.

**Migration path:** an API key dependency on both FastAPI apps via
`Depends()`, or authentication terminated at a reverse proxy in front of
both services. Not scheduled — named as explicit future work. The gap
itself is unchanged by this document; only its documentation is.

## 3. Cross-repo dependency pinning policy

Policy: a cross-repo dependency pin must always reference a reviewed,
merged commit or tag on the target repo's `main`, never an active working
branch. Any change to a pin goes through the same phase-branch/PR process
as any other code change — no direct edits to a pin outside that process.

This isn't something a runtime test can enforce; it's a practice this
project holds to, verified here by inspection against every current
cross-repo pin, all four now commit-pinned, none floating:

- `genai-block6-multiagent/requirements.txt`'s `block5_agent` pin —
  previously `@main` (a floating branch reference, flagged as a gap in
  `docs/spec.md`'s LLM03 section), now a fixed commit.
- `genai-block8-capstone/requirements.txt`'s `block5_agent` pin — fixed
  commit.
- `genai-block8-capstone/requirements.txt`'s `block6_multiagent` pin —
  fixed commit.
- `genai-block8-capstone/docker/block4.Dockerfile`'s `BLOCK4_COMMIT`
  build arg — fixed commit.

## 4. Risk status

One line per primary risk from `docs/spec.md` §2, each pointing at the
specific test proving its blocked/flagged claim.

**LLM01 — Prompt Injection**

- Direct injection (caller-supplied `condition`/`lab`/`drug_a`/`drug_b`):
  *flagged*, not blocked — an out-of-vocabulary or injection-shaped value
  surfaces as a logged flag, the run still completes normally, control
  flow never changes. `genai-block5-agent/block5_agent/plausibility_check.py`,
  proven by `tests/test_plausibility_check.py` (10 tests) and
  `tests/test_agent_plausibility.py` (3 tests, including
  `test_implausible_condition_is_flagged_but_run_completes_normally`).
- Indirect injection, Path A (citations): *blocked*.
  `genai-block6-multiagent/scripts/citation_sanitization.py`, proven by
  `tests/test_citation_sanitization.py` (unit-level) and
  `tests/test_citation_hardening_e2e.py::test_planted_injection_is_neutralized_by_the_time_it_reaches_citations`
  (end-to-end, through the real ingestion → retrieval → citation
  pipeline).
- Indirect injection, Path B (Block 4's `generate_answer` prompt):
  *blocked*. `genai-block4-rag-eval/scripts/sanitize.py`, proven by
  `tests/test_sanitize.py` and
  `tests/test_generate.py::test_generate_answer_prompt_is_clean_for_planted_injection`
  (plus its `_with_newlines` variant).
- `MultiAgentAnswer.answer`/`.caveat` sanitization at every assignment
  site: *blocked*. `genai-block6-multiagent/tests/test_orchestrator.py`'s
  four sanitization tests, one per real construction site —
  `test_citation_snippets_are_sanitized_and_trimmed_when_constructed`,
  `test_clinical_result_answer_and_caveat_are_sanitized_in_clinical_only_degraded_mode`,
  `test_clinical_result_answer_is_sanitized_in_the_reconciled_path`, and
  `test_cohort_only_degraded_answer_sanitizes_caller_input_echoed_into_the_template`
  (the one caller-input-echo site with no LLM involved, found and fixed
  during this same review).

**LLM02 — Sensitive Information Disclosure**

- Data-layer (Neo4j RBAC): *flagged/documented*, not blocked — see §1
  above. No runtime test can close a Community Edition structural limit.
- Network/API-layer: *flagged/documented*, not blocked — see §2 above.
  Mitigated by deployment configuration, not application code; no
  application-level test could prove it either way.
- Field-layer over-exposure (citation trimming): *blocked*.
  `genai-block6-multiagent/scripts/citation_sanitization.py::trim_citation_snippet`,
  proven by
  `tests/test_citation_sanitization.py::test_keeps_only_sentences_containing_a_query_term`.

**LLM03 — Supply Chain**

*Flagged*, not blocked — a practice, not something a runtime test
catches. See §3 above for the policy statement and current compliance
status across all four cross-repo pins.

**LLM06 — Excessive Agency**

- Orchestrator state tampering: *blocked*.
  `genai-block6-multiagent/scripts/state_validation.py::validate_state_update`,
  proven by `tests/test_state_validation.py` (9 tests).
- Cypher injection on the Cohort Enumeration Agent's query: *blocked*.
  `genai-block6-multiagent/tests/test_cohort_tool.py`'s four adversarial
  tests (`test_adversarial_condition_value_is_bound_as_a_parameter_never_interpolated`,
  `test_adversarial_condition_value_never_makes_the_query_writable`,
  `test_adversarial_lab_is_rejected_before_reaching_the_driver`,
  `test_adversarial_comparison_is_rejected_before_reaching_the_driver`) —
  matching Block 5's own equivalent three-test coverage in
  `genai-block5-agent/tests/test_graph_tool.py`, same rigor on both
  agents' Cypher tools, not inherited assurance.
- Reconciliation error-handling gap (`_reconcile_error_answer` itself
  raising): *blocked*.
  `genai-block6-multiagent/tests/test_orchestrator.py::test_reconcile_error_answer_itself_raising_still_returns_a_valid_answer`.

**LLM08 — Vector and Embedding Weaknesses**

Query/write path key separation: *blocked* in code — the query-path
Pinecone key can no longer write or delete regardless of what code runs
against it, proven by
`genai-block4-rag-eval/tests/test_pinecone_key_scope.py`'s
`test_query_scoped_key_cannot_upsert` and
`test_query_scoped_key_cannot_delete` (live-credential tests, skipped
without them). One operational step remains outside code: the actual
`PINECONE_QUERY_API_KEY` (`DataPlaneViewer`, query-only) still needs
creating in the Pinecone console and setting in `.env` — this phase wired
the code to expect and enforce a scoped key, not created the key itself.

**LLM10 — Unbounded Consumption**

- Zero-delay retries (Block 5): *blocked*.
  `genai-block5-agent/tests/test_error_classification.py` (20 tests) plus
  `tests/test_agent_answers.py`'s retry-timing tests
  (`test_answer_step_failed_after_one_retry_returns_fixed_answer`,
  `test_answer_step_retryable_failure_succeeds_on_second_attempt`,
  `test_answer_step_permanent_failure_fails_immediately_without_retrying`,
  `test_answer_step_unclassified_failure_fails_immediately_without_retrying`).
- Unbounded `vocabulary_check.py` query: *blocked*.
  `genai-block6-multiagent/tests/test_vocabulary_check.py::test_both_queries_use_the_shared_graph_query_timeout`
  and `test_a_genuinely_slow_query_surfaces_as_a_timeout_not_a_hang`.
- Cohort agent's exhaustive enumeration query cost: *flagged*, not
  capped, by design (a hard limit would reintroduce the undercounting
  problem this agent exists to solve).
  `genai-block6-multiagent/tests/test_cohort_tool.py`'s soft-alert
  logging tests
  (`test_query_full_cohort_warns_when_result_exceeds_the_soft_alert_patient_threshold`,
  `test_count_drugs_exhaustive_warns_when_cohort_size_exceeds_the_soft_alert_threshold`,
  `test_slow_query_logs_a_runtime_soft_alert`).
- Non-finite float (`inf`/`nan`) crash on rejection: *blocked*, in both
  repos that have their own HTTP layer.
  `genai-block4-rag-eval/tests/test_api.py::test_value_infinity_returns_422`
  and `test_value_nan_returns_422`; identically,
  `genai-block8-capstone/tests/test_api.py::test_query_rejects_infinity_value`
  and `test_query_rejects_nan_value`.
