# AI-in-Prod Patterns (Educational Project)

## Overview

Large language models (LLMs) can speed up drafting, code suggestions, and internal automation. In production workflows, the same qualities that make LLMs useful (fast, fluent, confident outputs) can introduce reliability issues:

* Confident but incorrect answers (“looks right” failures)
* Subtle edge-case bugs in generated code
* Automation complacency (outputs accepted without review)
* Silent regressions when prompts/models/tools change
* Security risks when LLM output is used as commands, SQL, or tool calls

This README describes a practical, engineering-first workflow that treats AI output as untrusted input and adds deterministic controls (schemas, tests, policies), risk-tiered human oversight, and evaluation to prevent drift. The goal is to keep AI as leverage, not technical debt.

---

## Step-by-step workflow

### Step 1: Classify the use case by risk tier

Use a tier model so controls match impact.

Tier 0 (Low risk)

* Examples: brainstorming, first drafts, internal notes
* Controls: human edit before publishing; no direct actions

Tier 1 (Medium risk)

* Examples: code suggestions, refactors, internal scripts
* Controls: PR review required; CI gates must pass (lint/typecheck/tests)

Tier 2 (High risk)

* Examples: production automation, customer-facing decisions, permissions, destructive actions
* Controls: strict validation + approval gate + sandboxed execution + rollout controls

Rule of thumb: If the output can affect customers, security, money, permissions, or production state, treat it as Tier 2.

---

### Step 2: Constrain outputs with explicit contracts

When AI output feeds downstream systems, prefer structured outputs and enforceable contracts.

Example output contract (shape)

* task: string
* summary: string
* risk_flags: string[]
* actions: { name: string, params: object, reason: string }[]

Key constraints

* Output must be strictly parseable (JSON or another strict format).
* Required fields must always be present.
* Any action must be explicit and separately validated.

---

### Step 3: Validate deterministically (fail closed)

Avoid using “LLM self-check” as your primary safety mechanism. Use deterministic validators first:

* Schema validation (JSON Schema / Zod / Pydantic)
* Parsing/compilation/type-checking for code outputs
* Linters and formatters
* Unit + integration tests
* Policy checks (allowlists, bounds, invariants)
* Security scanning (SAST/dependency scanning) where applicable

Fail closed: if validation fails, reject the output and require correction.

Pseudocode: generation + validation gate

```pseudo
input = request()

tier = classify(input)

raw = llm.generate(prompt_for(input, tier))

parsed = strict_parse(raw)
if not parsed:
  raw = llm.generate(prompt_for(input, tier) + "Return strict JSON only.")
  parsed = strict_parse(raw)

assert schema_validate(parsed)
assert policy_validate(parsed, tier)

if tier == HIGH:
  require_human_approval(parsed)

if parsed.contains_actions:
  for action in parsed.actions:
    assert action.name in ALLOWLIST
    assert params_within_bounds(action.params)
  run_in_sandbox(parsed.actions)

record_audit_log(input, raw, parsed, tier)

return parsed
```

---

### Step 4: Apply human oversight where it reduces risk

Human-in-the-loop works best when it’s targeted:

* Tier 0: human edits before publishing externally
* Tier 1: PR review + CI gates before merge
* Tier 2: explicit approval before executing actions or shipping customer-impacting behavior

Avoid mandatory approvals for low-risk tasks; it increases bypass behavior. Keep Tier 2 strict and automate Tier 0/1 validation.

---

### Step 5: Treat AI-generated code as “needs tests”

Edge-case bugs often slip in when AI-generated code is accepted based on plausibility.

A policy that scales:

* If AI authored or significantly modified code, require:

  * one happy-path unit test
  * one edge-case test (or property-based tests for pure functions)
* CI must run:

  * lint/format
  * type checks
  * unit tests
  * integration tests (when relevant)
  * security scans (when relevant)

Reviewer checklist (fast)

* input validation and error handling
* timeouts/retries/backoff for I/O
* authn/authz boundaries
* concurrency and shared-state assumptions
* backward compatibility and migration impact
* observability for new behavior (logs/metrics/traces)

---

### Step 6: Control automation and tool calling (least privilege + sandbox)

If the model can call tools or trigger actions, treat it like an automation system.

Controls that consistently reduce incidents:

* Least privilege: read-only tools by default, separate read/write capabilities
* Allowlisted actions only (no arbitrary shell/SQL)
* Bounded parameters: rate limits, max recipients, max rows, timeouts
* Sandboxed execution: restricted filesystem/network, no direct prod credentials
* Approval gate for destructive/irreversible actions
* Audit logs: prompts, outputs, tool calls, approvals

Allowlist concept (example)

```pseudo
ALLOWLIST = {
  "create_ticket": ["title", "body", "labels"],
  "search_docs": ["query", "limit"],
  "generate_report": ["date_range"]
}

deny_if_action_not_in(ALLOWLIST)
deny_if_params_not_subset_of(ALLOWLIST[action])
```

---

### Step 7: Prevent silent drift with evals and monitoring

Reliability degrades as prompts evolve, models change, and dependencies shift.

Minimum viable evaluation setup:

* Maintain a golden set of real tasks (10–50 cases)
* Rerun on:

  * prompt changes
  * model/version changes
  * major dependency changes
  * on a schedule (weekly/monthly)
* Track:

  * pass rate / regressions
  * unsafe-output rate
  * latency and cost budgets

Pseudocode: eval harness

```pseudo
for tc in golden_set:
  out = llm.generate(tc.prompt)
  assert schema_validate(out, tc.schema)
  assert policy_validate(out, tc.policy)
  assert tc.scorer(out) >= tc.threshold
report_metrics()
```

---

## Architecture overview

Typical safe-by-default pipeline:

```text
Client / CI / App
  -> Policy Router (risk tier + data/tool access rules)
  -> LLM (prompt includes output contract + constraints)
  -> Validation Layer
     - schema validation
     - deterministic checks (tests/typecheck/policies)
     - optional secondary critique (non-blocking)
  -> Approval Gate (Tier 2)
  -> Action Runner
     - allowlisted actions only
     - sandboxed execution
  -> Audit + Observability
     - redacted logs
     - metrics (pass rate, regressions, cost/latency)
```

Key idea: the LLM generates candidates; the surrounding system enforces correctness and safety.

---

## Installation / Setup

This is an educational reference. You can adopt it in two ways.

### Option A: Process-only adoption (fastest)

1. Add a PR template checkbox: “AI-assisted changes?”
2. Adopt the risk-tier model in your engineering handbook.
3. Enforce Tier 1 CI gates for merges (lint/typecheck/tests).
4. Require Tier 2 approval gate for any AI-driven action that changes production state.
5. Create a small golden eval set and run it on prompt/model changes.

### Option B: Minimal validation wrapper (recommended for automation)

Implement a wrapper service/module that provides:

* strict parsing (no best-effort)
* schema validation for structured outputs
* policy allowlists and bounds checks
* sandbox action runner for tool execution
* audit logging with redaction

Stack examples:

* TypeScript: Zod + ESLint + tsc + Jest
* Python: Pydantic + pytest + ruff/mypy
* Go: strict JSON decode + go test + linters

---

## Examples

### Example 1: Content drafting (Tier 0)

Workflow:

1. Model produces outline + claim list
2. Human reviews claims and removes/flags uncertain statements
3. Publish only after review

Prompt constraints:

* “List claims separately”
* “Use ‘unknown’ when uncertain”
* “Do not invent citations or links”

### Example 2: Code suggestions (Tier 1)

Workflow:

1. Model proposes patch plus tests
2. CI runs typecheck + tests
3. Reviewer checks boundaries and edge cases
4. Merge only when tests cover behavior and edge conditions

### Example 3: Internal automation with tool calling (Tier 2)

Workflow:

1. Model outputs structured plan with explicit actions
2. Schema + policy validation
3. Human approval
4. Actions execute in sandbox with allowlisted tools
5. Everything is recorded in audit logs

---

## Recommended Tools

This workflow can be implemented with common OSS components, or supported by a managed platform.

Building blocks (common):

* Output contracts and schema validation:

  * JSON Schema
  * Zod (TypeScript)
  * Pydantic (Python)
* Deterministic verification:

  * unit/integration tests (pytest, Jest, JUnit, go test)
  * linters/type checkers (eslint, tsc, mypy, ruff, golangci-lint)
  * security scanning (SAST/dependency scanners, where relevant)
* Rollout safety:

  * feature flags / canary releases
  * rate limiting / budgets / timeouts
  * central logging/metrics

Managed option (optional, easiest path):

* {Product Name}

  * Use case: a managed layer to centralize prompt/version management, enforce structured outputs, run evals/regression checks, and maintain audit logs/RBAC in one place.
  * This repository does not require it. It’s an option if you prefer not to build and operate the “glue” layer yourself.

Evaluation checklist for {Product Name} (or any managed option):

* Can you enforce structured outputs and validate schemas?
* Can you restrict tool/action access via allowlists and least privilege?
* Are audit logs and redaction controls available?
* Can you run evals/regression tests on prompt/model changes?
* Does it integrate with CI/CD and your observability stack?

---

## FAQ

Q: Should we ban AI for production code?
A: Bans are hard to enforce. A more workable approach is disclosure (AI-assisted) plus CI gates and tests for AI-generated changes.

Q: Can we use an LLM to validate another LLM?
A: As a secondary signal, yes. For primary gating, deterministic checks (schema/tests/policy) are more reliable.

Q: How do we stop teammates from accepting AI outputs without review?
A: Make verification explicit: PR checkbox, required tests, and a short “what I verified” note. Keep Tier 2 behind approvals.

Q: What about prompt injection?
A: If the model reads untrusted text or can call tools, separate instructions from data, restrict tools to allowlists, sandbox execution, and log tool calls.

---

## Troubleshooting

Issue: Output parses but is still incorrect

* Strengthen deterministic checks (invariants, tests, policy rules).
* Reduce task scope and ambiguity.
* Require “unknown” behavior for uncertain facts in content/decision prompts.

Issue: Developers bypass controls

* Keep Tier 0/1 friction low (templates, automation, defaults).
* Keep Tier 2 strict (approval + sandbox + rollout controls).
* Enforce gates via CI rather than relying on discipline.

Issue: Costs/latency are unstable

* Use risk tiers to reserve larger models for high-value tasks.
* Add budgets/timeouts and caching for stable queries.
* Set step limits for agentic flows.

---

## Summary and next steps

Summary:

* Classify AI usage by risk tier.
* Constrain outputs with contracts.
* Validate with deterministic gates (schema/tests/policies).
* Apply targeted human oversight for high-impact actions.
* Prevent drift with evals and monitoring.
* Treat prompts and validation like code (versioned, reviewed, owned).

Next steps:

1. Add an “AI-assisted changes?” checkbox to PR templates.
2. Enforce CI gates for Tier 1 code paths (tests/typecheck/lint).
3. Implement schema validation for any AI output consumed by systems.
4. Put Tier 2 automation behind approval gates and sandboxed execution.
5. Create a golden eval set and run it on prompt/model changes.
