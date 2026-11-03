# 02 — Support Ticket Triage Agent

> **Documentation-only project blueprint.** This folder intentionally contains only this README. The system described below has not been implemented, deployed, or certified as production ready. Production readiness is the build objective; the acceptance gates are proposed targets to validate, not measured results.

| Learning position | Difficulty | New engineering challenge |
| --- | --- | --- |
| 2 of 20 | Foundation+ | Confidence-aware routing and human correction |

[Previous: Task Planning Agent](../01-task-planning-agent/README.md) · [Next: Personal Document Research Agent](../03-personal-document-research-agent/README.md)

## Product idea and intended users

Help a small SaaS support team classify incoming tickets, retrieve routing policies, detect missing information, and prepare a response for a human reviewer.

**Example end-to-end use case:** A customer reports being locked out after changing their email. The agent looks up the account-access policy, proposes the correct queue and urgency, and drafts a clarification without requesting passwords.

**Initial boundary:** Draft replies only; no refunds, password changes, or automatic customer messages.

## Why this is an agent

The model chooses the next permitted tool based on observations and checks whether the evidence is sufficient before finishing. The application owns authorization, persistence, validation, and execution limits. A sequence of prompts alone is not the goal.

**Workflow:** Receive ticket → redact secrets → consult policy → classify → check evidence and required fields → draft reply → reviewer accepts or corrects.

**Tool contracts:** lookup_routing_policy(product, category), find_similar_resolved_tickets(query), validate_ticket_fields(ticket). Use synthetic ticket history first.

Implement each tool as a typed interface with validated input, a timeout, a scoped execution context, and a typed result containing evidence references or an explicit error. Never accept executable code or an unrestricted tool name directly from the model. Give the agent only the tools needed for its current stage.

## Prerequisites and suggested architecture

Know the earlier projects' tool schemas, bounded loops, test fixtures, and run logging. This is a standalone build: reuse the concepts from projects 01–01, without coupling their source trees.

**Suggested stack:** Python, FastAPI, Pydantic, PostgreSQL; a simple React review UI.

These are design options, not required dependency versions. Select supported releases when implementation begins, verify their official documentation, and commit a lockfile. Use a small compatible local instruction model or a hosted model through the same adapter; measure whether its structured output and tool-selection quality meet the evaluation gates.

Keep the request handler, agent coordinator, model adapter, tool adapters, domain validation, and storage separate. Begin as a modular service. Add workers only when ingestion, long execution, or recovery requires them; do not split into microservices merely for appearance.

**Proposed modules to create later:**

- `api/`: authentication, input schemas, run creation, review actions, and result delivery.
- `domain/`: business objects, deterministic invariants, and state transitions.
- `agent/`: versioned instructions, loop/coordinator, stopping rules, and model adapter.
- `tools/`: scoped integrations with typed requests and responses.
- `storage/`: repositories, migrations, and run checkpoints.
- `evals/` and `tests/`: labeled fixtures, adversarial cases, and regression checks.
- `ops/`: container configuration, dashboards, deployment notes, and recovery runbooks.

This is a proposed future layout, not folders included in this documentation-only contribution.

## Data model and service contract

**Domain records:** Ticket(id, tenant_id, external_id, body_hash, received_at); Recommendation(ticket_id, queue, urgency, policy_ids, missing_fields); Review(id, decision, corrected_labels).

Give persistent records stable IDs, ownership scope where applicable, timestamps, and schema versions. Separate source evidence from generated interpretations. Retain the model configuration, prompt version, tool versions, and policy version for each run without storing hidden model reasoning.

For a service implementation, start with:

| Endpoint | Purpose |
| --- | --- |
| `POST /runs` | Validate a domain-specific request and create a bounded run; return a run ID and status. |
| `GET /runs/{id}` | Return authorized status, evidence, result, usage, and explicit errors. |
| `POST /runs/{id}/cancel` | Request cancellation; stop before the next tool action and reconcile any in-flight effect. |
| `GET /health/live` | Process liveness without exposing secrets. |
| `GET /health/ready` | Readiness of dependencies required to accept work. |

Add review or approval endpoints only when the workflow needs them. Check resource ownership on every request and tool call. Use an idempotency key when accepting retries that could create duplicate domain records or actions.

Represent execution with explicit states such as `queued`, `running`, `needs_input`, `needs_review`, `completed`, `failed`, and `cancelled`; enable only states used by this project. Cancellation is not a promise that an external effect has been undone.

## Step-by-step implementation plan

1. **Milestone 1:** Define a stable taxonomy for five ticket categories and deterministic rules for security escalations. Create 100 synthetic labeled examples.
2. **Milestone 2:** Implement a policy lookup and a similar-ticket search over approved, sanitized historical records.
3. **Milestone 3:** Build the tool-calling triage loop with structured category, urgency, evidence IDs, and missing-information fields.
4. **Milestone 4:** Add a review inbox showing the ticket, cited policy, draft response, and correction controls.
5. **Milestone 5:** Accept webhook deliveries with signature verification and deduplicate by tenant and external ticket ID. Initially simulate the ticket provider.
6. **Milestone 6:** Measure label quality, calibrate the escalation threshold on held-out data, and deploy in shadow mode before enabling assignment.
7. **Release rehearsal:** Run the full evaluation suite against the pinned candidate, perform a restore exercise, inspect a redacted trace, and document supported inputs, known limitations, rollback, and operator ownership.

Complete and demonstrate each milestone before adding the next. Use synthetic fixtures and mock integrations until validation and authorization work reliably. The first vertical slice should exercise input → one real tool → validated output → saved run → repeatable test.

## Model and execution policy

Keep provider credentials in server-side secret configuration. Never place them in source control, browser bundles, prompts, or logs. Model choice is configurable per role: extraction, planning, synthesis, embeddings, and judging need not share one model. Only add roles actually needed by this project.

Start with a configurable ceiling of 8 tool actions, 1 schema-repair attempt, and a 120-second interactive run deadline. Long-running projects must checkpoint and use separately configured job deadlines and per-stage ceilings. These are initial limits to tune through measurement, not universal performance guarantees.

Set input/output token limits and a maximum estimated run cost before invoking a model. Track actual usage and stop or request review when the budget is exhausted. Retry only transient failures with bounded exponential backoff and jitter; do not blindly retry side effects. Fail clearly when a chosen local model lacks the required capability instead of silently falling back to a paid provider.

## Failure cases and safeguards

Route unknown products and ambiguous categories to manual review. Urgency comes from policy and observed signals, not model confidence alone. Duplicate webhooks must not create duplicate tickets.

Across the implementation:

- Reject invalid model output at schema boundaries; after the bounded repair attempt, return an actionable error.
- Treat retrieved documents, transcripts, web pages, repository content, and tool output as untrusted evidence. They cannot grant permissions or override instructions.
- Use least-privilege credentials and server-enforced ownership checks. Keep sensitive data out of model context unless necessary and permitted.
- Preserve the original input and approved revisions according to a documented retention policy. Provide deletion for user-owned data and derived indexes where applicable.
- Show partial, stale, unsupported, or unverified results explicitly. Never convert a failed check into a success claim.

## Testing and measurable acceptance gates

**Project-specific evaluation:** Use a held-out set of at least 100 tickets. Target macro-F1 ≥0.85 for category routing, 100% recall on explicit security escalation fixtures, and zero automatic reply sends.

Use a development set for iteration and a held-out set for final evaluation. Save input fixtures, expected invariants, dataset versions, and human-review rubrics. Evaluate tool choice and task outcome separately from how fluent the final response sounds.

Required checks before release:

1. Unit-test deterministic domain rules and schema validation.
2. Contract-test model/tool adapters using recorded or synthetic responses, including malformed arguments, unknown tools, and rate limits.
3. Run end-to-end cases for the happy path, missing input, provider outage, cancellation, and restart where persistence exists.
4. Test prompt injection and unauthorized resource access. For effectful tools, prove approvals and idempotency under retries and timeouts.
5. Measure task success, groundedness where relevant, p50/p95 latency, tool failure rate, tokens, and cost per successful run on a documented workload.

The numeric targets above are starting release gates. Adjust them only with an explicit written rationale, not to hide a failing candidate.

## Deployment and operations checklist

- Build a reproducible container and run schema migrations as a controlled deployment step. Use separate development, staging, and production configuration.
- Apply authentication, request-size limits, rate limits, connection limits, and timeouts. Enable TLS at the ingress.
- Log request/run IDs, stage transitions, tool names, latency, usage, and error classes. Redact payloads and never log credentials or hidden reasoning.
- Add alerts for elevated failure rate, stalled runs, budget exhaustion, and queue depth if a worker is used. Set workload-specific SLOs after a staging baseline.
- Back up persistent state, test restore, define retention, and document how to rotate credentials and disable a malfunctioning integration.
- Release first to a small supported workload. Keep a rollback path for application, prompt, policy, and model configuration changes; check database backward compatibility.
- Record a release decision with the evaluated artifact version, remaining risks, owner, and evidence. Do not label the build production ready merely because the demo works.

## Portfolio demonstration and completion

**Demo scenario:** Compare a normal billing ticket, an ambiguous ticket, and a security incident in the review inbox.

Deliver the running narrow workflow, a short architecture explanation, reproducible evaluation results, a redacted trace, and an operator runbook. Explain at least one failed approach or tradeoff using measured evidence.

**Done means:** the supported use case works end to end; the project-specific gates and isolation checks pass; failure states are visible; recovery and rollback have been rehearsed; and a reviewer can reproduce the result. Stretch features come only after this baseline.

## Optional extensions

Add a second model adapter and compare quality, latency, and cost on the same held-out tasks. Extend the supported input domain only after the first domain meets its gates.

Keep all future code and deployment artifacts within this project's own folder when you implement it. This README is the build guide, not an implementation claim.
