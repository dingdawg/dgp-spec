# Decentralized Governance Protocol (DGP) — v1.0

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Spec Version](https://img.shields.io/badge/spec-v1.0-green.svg)](DGP-v1.0.md)

DGP is an open protocol for governing AI agent execution. It defines message schemas, a deterministic state machine, and a six-layer governance stack that wraps agent actions in a verifiable governance envelope.

DGP is to agent execution what TLS is to HTTP — a security and trust layer that wraps the underlying protocol without replacing it.

## What DGP Solves

- **No universal standard** — agents execute across MCP, A2A, OpenAI, and custom APIs with no shared governance model
- **Compliance requires proof** — EU AI Act (Aug 2, 2026) mandates machine-readable records, not behavioral promises
- **Trust can't be self-reported** — third-party verification requires a common protocol with deterministic computation

## Read the Spec

→ **[DGP-v1.0.md](DGP-v1.0.md)**

## Message Schemas (JSON Schema 2020-12)

| Schema | Purpose |
|--------|---------|
| [ActionRequest](schemas/ActionRequest.schema.json) | Agent action submitted for evaluation |
| [PolicyDecision](schemas/PolicyDecision.schema.json) | Output of policy evaluation |
| [ExecutionReceipt](schemas/ExecutionReceipt.schema.json) | Signed proof of governed execution |
| [AuditEntry](schemas/AuditEntry.schema.json) | Tamper-evident audit chain entry |

## Compliance Test Suite

→ [github.com/dingdawg/dgp-compliance-tests](https://github.com/dingdawg/dgp-compliance-tests)

268 tests. Pass all = DGP v1.0 compliant.

## Reference Packages

| Package | Language | Install |
|---------|----------|---------|
| `@dingdawg/dgp-types` | TypeScript | `npm install @dingdawg/dgp-types` |
| `dgp-types` | Python | `pip install dgp-types` |

## Transport Adapters

DGP is transport-agnostic. The spec defines mappings for MCP, A2A, OpenAI function calling, and x402 payment protocol.

## Compliance Mapping

| Regulation | DGP Coverage |
|------------|-------------|
| EU AI Act Art. 9, 12, 14 | Risk management, record-keeping, human oversight |
| Colorado AI Act | Impact assessments, notification, appeal |
| SOC 2 Type II | Access controls, monitoring, anomaly detection |
| HIPAA | Audit controls, integrity, authentication |

## License

Apache 2.0 — implement, extend, and distribute without restriction.
