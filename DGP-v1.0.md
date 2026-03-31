# Decentralized Governance Protocol (DGP) — Version 1.0

**Status:** Published Specification
**Version:** 1.0
**Date:** 2026-03-31
**License:** Apache 2.0

---

## Abstract

The Decentralized Governance Protocol (DGP) is an open specification for governing AI agent execution across any platform, transport, or runtime. DGP defines message schemas, a deterministic state machine, and a six-layer governance stack that wraps agent actions in a verifiable governance envelope. Any developer can implement a compliant governance layer from this specification alone. Any enterprise can independently verify compliance.

DGP is to agent execution what TLS is to HTTP: a security and trust layer that wraps the underlying protocol without replacing it. DGP is transport-agnostic — it operates over MCP, A2A, OpenAI function calling, x402, or raw HTTP.

This specification maps directly to EU AI Act Articles 9 (risk management), 12 (record-keeping), and 14 (human oversight), providing a structural compliance path for organizations deploying AI agents in regulated environments.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Architecture Overview](#2-architecture-overview)
3. [Core Objects](#3-core-objects)
4. [Message Schemas](#4-message-schemas)
5. [State Machine](#5-state-machine)
6. [Protocol Layers](#6-protocol-layers)
7. [Transport Adapters](#7-transport-adapters)
8. [Security Model](#8-security-model)
9. [Compliance Mapping](#9-compliance-mapping)
10. [Error Codes](#10-error-codes)
11. [Versioning Policy](#11-versioning-policy)
12. [Reference Implementation](#12-reference-implementation)
13. [Appendix A: Canonical JSON Specification](#appendix-a-canonical-json-specification)
14. [Appendix B: Test Vectors](#appendix-b-test-vectors)

---

## 1. Problem Statement

### 1.1 No Universal Governance Standard

AI agents execute across heterogeneous platforms — MCP servers, A2A task runners, OpenAI plugins, custom REST APIs — each with its own governance model, or none at all. Enterprise buyers deploying agents across multiple platforms cannot verify governance posture consistently. There is no shared language for expressing what an agent is allowed to do, proving what it did, or computing how trustworthy it is.

Regulatory pressure is accelerating. The EU AI Act (Regulation 2024/1689) takes effect August 2, 2026. The Colorado AI Act requires impact assessments for high-risk automated decisions. SOC 2 Type II audits increasingly require evidence of AI system controls. Organizations need a governance standard today — not after their first compliance audit fails.

### 1.2 Compliance Requires Structural Proof

Advisory governance — instructing an agent to "check before acting" — is unenforceable. Regulators require machine-readable records, not behavioral promises. EU AI Act Article 12 mandates automatic recording of events for high-risk AI systems with sufficient traceability throughout the system's lifetime. No existing protocol produces tamper-evident, hash-chained receipts for individual agent actions.

### 1.3 Trust Cannot Be Self-Reported

An agent reporting its own compliance status is unfalsifiable. Third-party verification requires a common protocol with deterministic computation. Trust scoring must be derived from observed behavior — receipts, audit chains, policy compliance rates — not declared capability. Cross-platform reputation portability requires a standard format that any verifier can interpret independently.

---

## 2. Architecture Overview

DGP defines a six-layer governance stack. Every governed action passes through all six layers in sequence.

```
Client (any agent runtime)
    |
    v
[Layer 1] Policy Evaluation        <-- Policy rules (JSON/YAML)
    |
    v
[Layer 2] Budget Gate               <-- Spend authorization
    |
    v
[Layer 3] Execution Wrapper          <-- Governance envelope
    |                                     (DGP does NOT execute)
    v
    Agent runtime executes the action
    |
    v
[Layer 4] Receipt Generation         <-- Signed proof of execution
    |
    v
[Layer 5] Audit Trail                <-- Hash-chained immutable log
    |
    v
[Layer 6] Trust Scoring              <-- Composite reputation (async)
```

### 2.1 Architectural Principles

1. **DGP is a layer, not a runtime.** It wraps actions; it does not execute them. DGP operates on top of MCP, A2A, OpenAI function calling, or any transport protocol.

2. **Layers 1-3 are pre-execution (synchronous).** The action does not proceed unless all three layers pass.

3. **Layer 4 is post-execution (synchronous).** Receipt generation blocks until complete — no unreceipted executions.

4. **Layers 5-6 are post-execution (asynchronous permitted).** Audit trail append and trust score update may be processed asynchronously but MUST complete within 60 seconds of receipt generation.

5. **Fail-closed by default.** If no policy matches an action, the action is DENIED. This is structural, not advisory.

6. **Stateless per-request, stateful per-chain.** Each DGP evaluation is a standalone request. Each receipt references the previous receipt's hash, forming a verifiable chain.


---

## 3. Core Objects

DGP defines six first-class objects.

### 3.1 Action

The atomic unit of agent behavior that DGP governs.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action_id` | string (UUID v4) | Yes | Globally unique identifier |
| `action_type` | string | Yes | Dot-notation hierarchy (e.g., `db.write.users`) |
| `agent_id` | string | Yes | Identifier of the executing agent |
| `resource` | string | Yes | Target of the action (URI, path, or identifier) |
| `parameters` | object | No | Action-specific payload (opaque to DGP) |
| `context` | object | No | Ambient context (session, user, environment) |
| `timestamp` | string (ISO-8601) | Yes | UTC timestamp set by the caller |

The `action_type` field uses dot-notation hierarchy to enable glob matching in policy rules (e.g., `db.*` matches all database actions).

### 3.2 Policy

A machine-readable governance rule.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `policy_id` | string | Yes | Unique identifier |
| `name` | string | Yes | Human-readable label |
| `effect` | enum | Yes | `"allow"`, `"deny"`, or `"flag"` |
| `priority` | integer | Yes | Lower number = evaluated first |
| `conditions` | object | Yes | Matching conditions |
| `metadata` | object | No | Optional policy metadata |

**Three-effect model:** The `"flag"` effect enables human oversight workflows. Flagged actions can proceed after human approval but auto-deny after a configurable TTL. This maps to EU AI Act Article 14 (human oversight measures).

### 3.3 Decision

The output of policy evaluation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `decision_id` | string (UUID v4) | Yes | Unique identifier |
| `action_id` | string (UUID v4) | Yes | References the evaluated Action |
| `status` | enum | Yes | `"APPROVED"`, `"FLAGGED"`, `"BLOCKED"` |
| `matched_policies` | string[] | No | Policy IDs that matched |
| `violations` | object[] | No | Policies that triggered deny/flag |
| `risk_score` | number (0.0-1.0) | Yes | Computed risk assessment |
| `reasoning` | string | No | Human-readable explanation |
| `requires_human_review` | boolean | No | True if FLAGGED |
| `budget_authorized` | boolean | No | True if Budget Gate passed |
| `spend_authorized` | string | No | Authorized spend amount |
| `timestamp` | string (ISO-8601) | Yes | When the decision was made |
| `expires_at` | string (ISO-8601) | Yes | Decision expiry (default: 60s TTL) |

**Decisions are time-bounded.** A stale governance decision cannot authorize execution. Default TTL: 60 seconds. This prevents replay attacks.

### 3.4 Receipt

Cryptographically signed proof that an action was executed under governance.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `receipt_id` | string | Yes | Format: `rcpt_` + 16 hex characters |
| `action_id` | string (UUID v4) | Yes | References the Action |
| `decision_id` | string (UUID v4) | Yes | References the authorizing Decision |
| `agent_id` | string | Yes | Who executed |
| `action_type` | string | Yes | What was executed |
| `resource` | string | Yes | What was affected |
| `outcome` | enum | Yes | `"success"`, `"failure"`, `"partial"` |
| `output_hash` | string (hex64) | No | SHA-256 of the action's output |
| `duration_ms` | integer | No | Execution time in milliseconds |
| `chain_hash` | string (hex64) | Yes | SHA-256 of canonical JSON of this receipt |
| `prev_hash` | string (hex64) | Yes | Previous receipt's chain_hash (genesis: 64 zeros) |
| `signature` | string | Yes | HMAC-SHA256 or ECDSA-P256, base64url encoded |
| `signature_algorithm` | enum | No | `"hmac-sha256"` (default) or `"ecdsa-p256"` |
| `rollback_available` | boolean | No | Whether the action can be reversed |
| `timestamp` | string (ISO-8601) | Yes | When the receipt was generated |

### 3.5 AuditRecord

An entry in the tamper-evident audit chain.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sequence_number` | integer | Yes | 1-based, monotonically increasing |
| `entry_id` | string (UUID v4) | Yes | Unique identifier |
| `receipt_id` | string | Yes | Links to the Receipt |
| `actor` | string | Yes | Who performed the action |
| `action` | string | Yes | What happened |
| `resource` | string | Yes | What was affected |
| `details` | object | No | Arbitrary metadata |
| `previous_hash` | string (hex64) | Yes | SHA-256 of the prior AuditRecord (genesis: 64 zeros) |
| `entry_hash` | string (hex64) | Yes | SHA-256 of this record (canonical JSON, sorted keys) |
| `timestamp` | string (ISO-8601) | Yes | When the entry was appended |

### 3.6 TrustScore

Composite agent reputation derived from observed behavior.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `subject` | string | Yes | Agent or account identifier |
| `score` | number (0-100) | Yes | Composite trust score |
| `grade` | enum | Yes | `"A"` (90+), `"B"` (80-89), `"C"` (70-79), `"D"` (60-69), `"F"` (<60) |
| `components` | object | Yes | Named components with individual 0-100 scores |
| `based_on` | integer | Yes | Number of receipts analyzed |
| `computed_at` | string (ISO-8601) | Yes | When the score was computed |
| `valid_until` | string (ISO-8601) | Yes | Score expiry (default: 24h of inactivity) |

**Standard component names:** `compliance_health`, `operational_stability`, `behavioral_consistency`, `capability_breadth`, `incident_history`.

The specification defines the TrustScore **format**. The **computation algorithm** (component weights, update functions, calibration) is implementation-specific.

---

## 4. Message Schemas

DGP defines four wire-format message schemas using JSON Schema 2020-12. Complete schema files are provided alongside this specification.

- `ActionRequest.schema.json`
- `PolicyDecision.schema.json`
- `ExecutionReceipt.schema.json`
- `AuditEntry.schema.json`

All schemas include a required `dgp_version` field set to `"1.0"`. All schemas set `additionalProperties: false` to prevent schema drift. Messages that fail schema validation MUST be rejected with error code `DGP-1901`.


---

## 5. State Machine

DGP defines a deterministic state machine with six states and strictly defined transitions.

### 5.1 State Diagram

```
                ┌──────────┐
                │ PENDING  │
                └────┬─────┘
                     │ evaluate()
                     v
                ┌──────────┐
          ┌─────┤EVALUATED ├─────┐
          │     └──────────┘     │
          │          │           │
     status=     status=      status=
     BLOCKED     FLAGGED     APPROVED
          │          │           │
          v          v           v
     ┌────────┐ ┌────────┐ ┌──────────┐
     │ DENIED │ │FLAGGED │ │ ALLOWED  │
     └────────┘ └───┬────┘ └────┬─────┘
                    │           │
           human_approve()      │
           human_deny()    execute()
           auto_expire()        │
                    │           v
                    v      ┌──────────┐
              ALLOWED or   │ EXECUTED │
              DENIED       └────┬─────┘
                                │ generate_receipt()
                                v
                           ┌──────────┐
                           │RECEIPTED │
                           └────┬─────┘
                                │ append_audit()
                                v
                           ┌──────────┐
                           │ AUDITED  │
                           └──────────┘
```

### 5.2 Transition Table

| From | To | Trigger | Guard Condition |
|------|----|---------|-----------------|
| PENDING | EVALUATED | `evaluate()` | Always — every action MUST be evaluated |
| EVALUATED | ALLOWED | automatic | `PolicyDecision.status == APPROVED` |
| EVALUATED | FLAGGED | automatic | `PolicyDecision.status == FLAGGED` |
| EVALUATED | DENIED | automatic | `PolicyDecision.status == BLOCKED` |
| FLAGGED | ALLOWED | `human_approve()` | Human approved within TTL |
| FLAGGED | DENIED | `human_deny()` | Human explicitly denied |
| FLAGGED | DENIED | `auto_expire()` | TTL expired without human action |
| ALLOWED | EXECUTED | `execute()` | `Decision.expires_at > now()` |
| EXECUTED | RECEIPTED | `generate_receipt()` | Receipt generated and signed |
| RECEIPTED | AUDITED | `append_audit()` | Audit entry appended to chain |

### 5.3 Terminal States

- **DENIED** is terminal. To retry, submit a new `ActionRequest` with a new `action_id`. Prevents replay attacks.
- **AUDITED** is terminal. The governance lifecycle is complete.

### 5.4 Timeouts

- **Decision expiry:** Default 60 seconds. An ALLOWED action not executed before expiry requires re-evaluation.
- **FLAGGED TTL:** Default 3600 seconds (1 hour). No human action within TTL auto-transitions to DENIED.

### 5.5 Invalid Transitions

Implementations MUST reject: PENDING->EXECUTED, PENDING->RECEIPTED, DENIED->ALLOWED, DENIED->EXECUTED, EXECUTED->PENDING, AUDITED->any, RECEIPTED->PENDING.

---

## 6. Protocol Layers

### 6.1 Layer 1: Policy Evaluation

**Input:** `DGP.ActionRequest` | **Output:** `DGP.PolicyDecision`

1. Load applicable policy set for the tenant/agent
2. Evaluate policies in priority order (lowest number first)
3. Match: `action_type` via glob, `resource` via substring, `agent_id` via glob, time window, risk tier
4. First matching policy determines effect (allow/deny/flag)
5. **If no policy matches: DENY.** Fail-closed, non-negotiable.
6. Compute `risk_score` on 0.0-1.0 scale (format standard, computation implementation-specific)
7. Set `expires_at` (default: now() + 60s)

### 6.2 Layer 2: Budget Gate

**Input:** `ActionRequest.budget` + tenant spend state | **Output:** `budget_authorized` + `spend_authorized`

1. No `budget` field: authorized at $0.00
2. Exceeds daily/monthly tenant budget: DENY
3. Exceeds per-action policy ceiling: DENY
4. Otherwise: authorize lesser of (requested, ceiling, remaining)

When transport is x402, Budget Gate maps to x402 payment negotiation.

### 6.3 Layer 3: Execution Wrapper

1. Verify `PolicyDecision.status == ALLOWED` and not expired
2. Generate execution context: `{ decision_id, action_id, authorized_at, expires_at, spend_authorized }`
3. Pass to agent runtime — **DGP does not execute; the runtime does**
4. Capture result: outcome, output artifact, duration_ms
5. Compute `output_hash = SHA-256(output artifact)`

### 6.4 Layer 4: Receipt Generation

1. Retrieve `prev_hash` (most recent receipt or genesis sentinel `"0" * 64`)
2. Assemble receipt body (action_id, decision_id, agent_id, action_type, resource, outcome, output_hash, duration_ms, prev_hash, timestamp)
3. `chain_hash = SHA-256(canonical_json(receipt_body))`
4. `receipt_id = "rcpt_" + SHA-256(chain_hash + timestamp)[:16]`
5. `signature = HMAC-SHA256(chain_hash, key)` or `ECDSA-P256(chain_hash, key)`, base64url
6. Return `DGP.ExecutionReceipt`

### 6.5 Layer 5: Audit Trail

1. `sequence_number` = max(existing) + 1, or 1 for genesis
2. `previous_hash` from last AuditEntry or genesis sentinel
3. Assemble audit body (sequence_number, timestamp, actor, action, resource, details, previous_hash)
4. `entry_hash = SHA-256(canonical_json(audit_body))`
5. Persist atomically, return `DGP.AuditEntry`

**Chain Verification:** Recompute entry_hash from fields, verify `entry.previous_hash == prior.entry_hash`, genesis must have `previous_hash == "0" * 64`.

### 6.6 Layer 6: Trust Scoring

**Interface (normative):** Score 0-100, grades A/B/C/D/F, five standard components, baseline 50.0 for new agents, expires after 24h inactivity.

**Algorithm (implementation-specific):** Component weights, update functions, cross-agent calibration, and pattern influence are left to implementations. Any algorithm producing scores in the defined format is compliant.

---

## 7. Transport Adapters

### 7.1 MCP (Model Context Protocol)

| MCP Tool | DGP Layer | Input | Output |
|----------|-----------|-------|--------|
| `dgp.evaluate` | Layer 1+2 | ActionRequest | PolicyDecision |
| `dgp.execute` | Layer 3+4 | { decision_id, action_result } | ExecutionReceipt |
| `dgp.receipt` | Layer 4 query | { receipt_id } | ExecutionReceipt |
| `dgp.audit` | Layer 5 query | { receipt_id } or { range } | AuditEntry[] |
| `dgp.trust` | Layer 6 query | { agent_id } | TrustScore |
| `dgp.status` | Meta | {} | { version, health, quota } |

### 7.2 A2A (Agent-to-Agent Protocol)

DGP maps to A2A task annotations with types `dgp/policy-decision` and `dgp/execution-receipt`.

### 7.3 OpenAI Function Calling

Agent calls `dgp_evaluate` before and `dgp_receipt` after function execution.

### 7.4 x402 Payment Protocol

DGP Layer 2 maps to x402 headers: `X-DGP-Decision-Id`, `X-DGP-Spend-Authorized`, `X-DGP-Receipt-Id`, `X-Payment`.


---

## 8. Security Model

### 8.1 Cryptographic Primitives

| Purpose | Algorithm | Notes |
|---------|-----------|-------|
| Receipt/audit chain hash | SHA-256 | Deterministic, canonical JSON |
| Receipt signing (basic) | HMAC-SHA256 | Shared secret, single-org |
| Receipt signing (enterprise) | ECDSA P-256 | Asymmetric, third-party verifiable |
| Receipt ID derivation | SHA-256 | `rcpt_` + first 16 hex chars |
| API key storage | Argon2id | Never stored plaintext |

### 8.2 Token Lifecycle

Creation (scopes + tier + 90-day expiry) -> Validation (revocation, expiry, quota, scope) -> Usage tracking (counter per action) -> Rotation (revoke old, create new) -> Revocation (immediate, idempotent, irreversible).

### 8.3 Rate Limiting

Per-key: 10 req/s burst, 5 req/s sustained. Per-tenant: daily quota by tier. Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.

### 8.4 Structural Security Properties

1. **Fail-closed:** No policy match = DENY
2. **Decision expiry:** Prevents replay attacks
3. **Chain integrity:** Tampering detectable by hash recomputation
4. **Separation of concerns:** DGP evaluates and records; never executes
5. **No self-governance:** Agents MUST NOT modify own trust score, policies, or audit chain

---

## 9. Compliance Mapping

### 9.1 EU AI Act (Regulation 2024/1689)

| Article | Requirement | DGP Coverage |
|---------|-------------|--------------|
| Art. 9 | Risk management system | Layer 1 + risk_score |
| Art. 9(2)(a) | Identify known/foreseeable risks | Policy conditions |
| Art. 9(2)(b) | Estimate and evaluate risks | risk_score (0.0-1.0) |
| Art. 9(7) | Testing for conformity | State machine prevents unevaluated execution |
| Art. 12 | Record-keeping / automatic logging | Layer 5 audit trail |
| Art. 12(2) | Log period, reference DB, input | AuditEntry fields |
| Art. 12(3) | Traceability throughout lifetime | Hash chain from genesis |
| Art. 14 | Human oversight | FLAGGED + requires_human_review + TTL |
| Art. 14(4)(d) | Decide not to use AI system | BLOCKED status |
| Art. 14(4)(e) | Intervene or interrupt | FLAGGED -> human_deny() -> DENIED |

### 9.2 Colorado AI Act (SB21-169)

Impact assessments via risk_score history. Notification via reasoning field. Appeal via FLAGGED state.

### 9.3 SOC 2 Type II

CC6.1 (access controls via tokens), CC7.1 (monitoring via audit+trust), CC7.2 (anomaly via trust degradation), CC8.1 (change mgmt via policy versioning).

### 9.4 HIPAA

164.312(b) audit controls, 164.312(c)(1) integrity via SHA-256 chain, 164.312(d) authentication via tokens.

---

## 10. Error Codes

### 10.1 Format

```json
{
  "error": {
    "code": "DGP-1001",
    "category": "POLICY",
    "message": "Action blocked by policy evaluation",
    "details": {},
    "retry_after": null
  }
}
```

### 10.2 Registry

| Range | Category |
|-------|----------|
| DGP-1000-1099 | POLICY |
| DGP-1100-1199 | BUDGET |
| DGP-1200-1299 | EXECUTION |
| DGP-1300-1399 | RECEIPT |
| DGP-1400-1499 | AUDIT |
| DGP-1500-1599 | TRUST |
| DGP-1600-1699 | AUTH |
| DGP-1700-1799 | RATE |
| DGP-1800-1899 | TRANSPORT |
| DGP-1900-1999 | PROTOCOL |

### 10.3 Defined Codes

| Code | Message | Retryable |
|------|---------|-----------|
| DGP-1001 | Action blocked by policy | No |
| DGP-1002 | No matching policy — default deny | No |
| DGP-1003 | Decision expired | Yes |
| DGP-1010 | Flagged — awaiting human review | Yes (after decision) |
| DGP-1100 | Budget exceeded | No (until reset) |
| DGP-1101 | Daily quota exhausted | No (until reset) |
| DGP-1200 | Decision not found or expired | Yes |
| DGP-1201 | Decision status not ALLOWED | No |
| DGP-1300 | Receipt chain integrity error | No |
| DGP-1301 | Signing key unavailable | Yes |
| DGP-1400 | Audit chain tampering detected | No |
| DGP-1500 | Trust score computation failed | Yes |
| DGP-1600 | Invalid or expired API key | No |
| DGP-1601 | Insufficient scope | No |
| DGP-1700 | Rate limit exceeded | Yes (after retry_after) |
| DGP-1900 | Unsupported DGP version | No |
| DGP-1901 | Invalid message schema | No |

---

## 11. Versioning Policy

**Format:** `MAJOR.MINOR` — no patch versions for protocol specs.

**MINOR:** Additive only (new optional fields, error codes, adapters). Existing implementations MUST NOT break.

**MAJOR:** Breaking changes. Previous major supported 12 months minimum.

**Negotiation:** Every message includes `dgp_version: "1.0"`. Reject unsupported major (DGP-1900). Accept higher minor, ignore unknown fields.

**Governance:** RFC process (propose -> review -> vote -> publish). 30-day comment for MINOR, 90-day for MAJOR.

---

## 12. Reference Implementation

### 12.1 Building Your Own

Minimum viable DGP implementation requires:

1. **Policy Engine:** Load JSON/YAML policies, evaluate in priority order, fail-closed
2. **Receipt Generator:** SHA-256 hash chain with canonical JSON, `rcpt_` ID format
3. **Audit Store:** Append-only with hash chain linking and genesis block
4. **State Machine:** Enforce valid transitions (Section 5), reject all others

Estimated effort: 2-3 days in any language with JSON and SHA-256.

### 12.2 Compliance Test Suite

Published test suite containing: schema validation, state machine transitions (valid + invalid), hash chain computation (known vectors), receipt ID derivation, genesis block handling.

Any implementation passing the compliance suite is DGP v1.0 compliant.

### 12.3 Reference Packages

- `@dingdawg/dgp-types` (npm) — TypeScript types from JSON schemas
- `dgp-types` (PyPI) — Pydantic models from JSON schemas

---

## Appendix A: Canonical JSON Specification

### Rules

1. Sort keys lexicographically (Unicode code point order)
2. Compact separators: `","` and `":"`  (no whitespace)
3. ASCII encoding: non-ASCII escaped as `\uXXXX`
4. UTF-8 encoding before hashing
5. No trailing newline

### Reference (Python)

```python
import hashlib, json

def canonical_json(obj: dict) -> str:
    return json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=True)

def compute_hash(obj: dict) -> str:
    return hashlib.sha256(canonical_json(obj).encode("utf-8")).hexdigest()
```

### Reference (TypeScript)

```typescript
import { createHash } from 'crypto';

function canonicalJson(obj: Record<string, unknown>): string {
  return JSON.stringify(obj, Object.keys(obj).sort());
}

function computeHash(obj: Record<string, unknown>): string {
  return createHash('sha256').update(canonicalJson(obj), 'utf8').digest('hex');
}
```

---

## Appendix B: Test Vectors

### B.1 Canonical JSON

**Input:** `{"zebra":1,"alpha":"hello","nested":{"b":2,"a":1}}`
**Expected:** `{"alpha":"hello","nested":{"a":1,"b":2},"zebra":1}`

### B.2 Genesis Receipt

Receipt body with `prev_hash` = 64 zeros. Compute `chain_hash` via canonical JSON + SHA-256. Derive `receipt_id` = `"rcpt_" + SHA-256(chain_hash + timestamp)[:16]`.

### B.3 Chain Verification

For any chain: recompute every entry_hash, verify each previous_hash links to prior entry_hash. Genesis previous_hash MUST be 64 zeros. Any mismatch = tampering.

---

## License

This specification is published under the Apache License 2.0. Implement, extend, and distribute DGP-compliant software without restriction.

All cryptographic algorithms referenced (SHA-256, HMAC-SHA256, ECDSA P-256, Argon2id) are public-domain or openly standardized. All regulatory citations reference publicly available law.

---

*DGP v1.0 — Published 2026-03-31*
