# SSI Protocol Specification

**Sovereign Synthetic Intelligence**
Version 1.0 | Status: Stable

---

## Abstract

The Sovereign Synthetic Intelligence (SSI) Protocol defines a governance and verification framework for autonomous AI systems. SSI provides cryptographically auditable decision trails, layered policy enforcement, and deterministic safety boundaries — enabling organizations to govern AI agents without trusting any single implementation, vendor, or model.

This document is the normative specification for SSI Protocol v1.0. All conformant implementations MUST satisfy the requirements defined herein. Supporting documentation is located in `/docs/`.

---

## Status of This Document

**Version:** 1.0
**Stability:** Stable
**Supersedes:** v0.3.1 (draft)
**Copyright:** DeAlgo Foundation. Published under CC BY 4.0.

The keywords MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## Table of Contents

1. Introduction
2. Terminology
3. System Architecture
4. Protocol Primitives
   - 4.1 Governance Envelopes
   - 4.2 RPX Record
   - 4.3 Decision Types
5. Component Specifications
   - 5.1 SSI Kernel
   - 5.2 SSI Gateway
6. Policy Evaluation Model
7. Security Model
8. Conformance
9. Versioning
10. References

---

## 1. Introduction

AI systems making high-impact decisions — in finance, infrastructure, healthcare, and other regulated domains — currently lack a standardized mechanism for verifiable governance. Audit trails are proprietary, policy enforcement is implementation-specific, and there is no interoperable way for regulators, operators, or third parties to verify that a system behaved as governed.

The SSI Protocol addresses this by defining:

- A **governance container format** (Governance Envelopes) for encoding policy as structured, signable documents
- An **audit record format** (RPX) that captures every decision with full provenance and cryptographic integrity
- A **decision evaluation model** that is deterministic, layered, and deny-by-default
- **Interface contracts** for the two primary protocol components: the Kernel and Gateway

SSI is implementation-agnostic. Any system that satisfies the conformance requirements in Section 8 is a valid SSI implementation.

---

## 2. Terminology

**Agent** — An autonomous AI system or service that requests permission to execute actions.

**Action** — A discrete, potentially irreversible operation that an Agent wishes to perform (e.g., a financial trade, a configuration change, a data write).

**Governance Envelope** — A versioned, signed policy container that defines the rules under which an Agent may operate. See Section 4.1.

**RPX Record** — A Request-Permission-Execution record. The atomic unit of SSI's audit trail. See Section 4.2.

**Kernel** — The policy evaluation engine. Receives permission requests, evaluates them against applicable Governance Envelopes, and emits a Decision with reasoning. See Section 5.1.

**Gateway** — The protocol entry point. Normalizes incoming requests from Agents, routes them to a Kernel, and returns the Decision. See Section 5.2.

**Decision** — The Kernel's authoritative response to a permission request: one of `ALLOW`, `DENY`, `MODIFY`, or `ADVISORY`.

**Audit Chain** — The ordered, append-only sequence of RPX Records produced by a deployment. Provides full reconstruction of all governed actions.

**Constitutional Policy** — A policy rule that MUST NOT be violated under any circumstances. Violation results in an immediate `DENY`.

**Operational Policy** — A context-dependent rule that guides behavior. May result in `MODIFY` or `ESCALATE`.

**Safety Policy** — A protective rule that generates `ADVISORY` signals or halts execution under defined conditions.

**Conformant Implementation** — Any system that satisfies all MUST requirements in Section 8 of this specification.

---

## 3. System Architecture

SSI is a layered protocol. The canonical deployment topology is:

```
┌─────────────────────────────────────────┐
│  AI Agents / Models / Services          │
└────────────────────┬────────────────────┘
                     │  Action requests
┌────────────────────▼────────────────────┐
│  SSI Gateway                            │
│  Protocol normalization, routing        │
└────────────────────┬────────────────────┘
                     │  Normalized permission requests
┌────────────────────▼────────────────────┐
│  SSI Kernel                             │
│  Policy evaluation, decision engine     │
└────────────────────┬────────────────────┘
                     │  Decisions + RPX records
┌────────────────────▼────────────────────┐
│  Audit Chain (RPX Store)                │
│  Immutable, cryptographically signed    │
└─────────────────────────────────────────┘
```

Agents MUST NOT execute high-impact actions without first receiving a Decision from a Kernel. This is the foundational invariant of the protocol.

Implementations MAY deploy Kernel and Gateway as separate services or as a unified component, provided the logical separation of responsibilities is preserved.

---

## 4. Protocol Primitives

### 4.1 Governance Envelopes

A Governance Envelope is the primary mechanism for encoding policy in SSI. It is a structured, versioned, cryptographically signed document that defines the rules under which one or more Agents may operate.

#### 4.1.1 Structure

```yaml
envelope_id: "<uuid-v4>"
version: "<semver>"
issued_by: "<governance-authority-id>"
issued_at: "<ISO-8601>"

applies_to:
  agents: ["<agent-id>", ...]
  actions: ["<action-type>", ...]
  scopes: ["<scope>", ...]

policies:
  constitutional:
    - rule_id: "<id>"
      description: "<human-readable description>"
      condition: "<expression>"
      violation_action: "DENY"

  operational:
    - rule_id: "<id>"
      description: "<human-readable description>"
      condition: "<expression>"
      action: "ALLOW | MODIFY | ESCALATE"
      escalation_target: "<queue-id>"       # required if action is ESCALATE

  safety:
    - rule_id: "<id>"
      description: "<human-readable description>"
      condition: "<expression>"
      action: "ADVISORY"
      message: "<human-readable warning>"

metadata:
  authority: "<issuing body name>"
  approval_date: "<ISO-8601>"
  review_date: "<ISO-8601>"
  signature: "<Ed25519 signature over canonical envelope bytes>"
```

#### 4.1.2 Requirements

- Envelopes MUST be signed by a recognized governance authority using Ed25519 or stronger.
- Kernels MUST reject envelopes with invalid or unverifiable signatures.
- Envelopes MUST use semantic versioning. Breaking policy changes MUST increment the major version.
- Kernels MUST support multiple active envelope versions simultaneously to allow graceful transitions.
- Envelopes MUST define `applies_to` scope. Kernels MUST NOT apply an envelope to Agents or actions outside its declared scope.

#### 4.1.3 Policy Evaluation Order

When multiple envelopes apply to a request, the Kernel MUST apply the **most-restrictive-wins** rule: a `DENY` from any applicable envelope MUST result in a `DENY` Decision, regardless of other envelopes.

Undefined actions (no envelope explicitly permits them) MUST be denied by default.

---

### 4.2 RPX Record

The RPX (Request-Permission-Execution) Record is the atomic unit of the SSI audit trail. Every permission evaluation MUST produce exactly one RPX Record.

#### 4.2.1 Schema

```json
{
  "rpx_version": "1.0.0",
  "record_id": "<uuid-v4>",
  "timestamp": "<ISO-8601>",
  "agent_id": "<string>",

  "request": {
    "action_type": "<string>",
    "parameters": {},
    "context": {}
  },

  "permission": {
    "decision": "ALLOW | DENY | MODIFY | ADVISORY",
    "reasoning": ["<step>", "..."],
    "policy_refs": ["<envelope-id>", "..."],
    "constraints": {}
  },

  "execution": {
    "status": "SUCCESS | FAILURE | SKIPPED | PENDING",
    "result": {},
    "errors": []
  },

  "metadata": {
    "kernel_version": "<string>",
    "gateway_id": "<string>",
    "session_id": "<string>"
  },

  "signature": "<Ed25519 signature over canonical record bytes>"
}
```

#### 4.2.2 Requirements

- Kernels MUST emit an RPX Record for every permission evaluation, including denied requests.
- RPX Records MUST be cryptographically signed before being written to the Audit Chain.
- Records MUST be written to an append-only store. Mutation of persisted records is a conformance violation.
- The `reasoning` field MUST contain a human-readable trace of the policy rules that determined the Decision.
- The `policy_refs` field MUST list all envelope IDs that contributed to the Decision.
- Records MUST be retained for a minimum duration defined by the operator's governance policy, and MUST be accessible to authorized auditors.

---

### 4.3 Decision Types

| Decision   | Meaning |
|------------|---------|
| `ALLOW`    | The action is permitted as requested. Agent MAY proceed. |
| `DENY`     | The action is prohibited. Agent MUST NOT execute. |
| `MODIFY`   | The action is permitted subject to constraints in `permission.constraints`. Agent MUST apply constraints before executing. |
| `ADVISORY` | The action is permitted but a safety signal has been raised. Agent SHOULD surface this to the operator. |

A Decision of `DENY` is final. Agents MUST NOT execute the requested action after receiving `DENY`, regardless of retry logic or fallback paths.

---

## 5. Component Specifications

### 5.1 SSI Kernel

The Kernel is the normative decision engine of the SSI Protocol.

#### 5.1.1 Responsibilities

- Load and validate applicable Governance Envelopes for each request
- Evaluate the request against all applicable policy rules
- Compute a single authoritative Decision
- Generate a reasoning trace
- Emit a signed RPX Record to the Audit Chain

#### 5.1.2 Processing Pipeline

```
Request received
      │
      ▼
Load applicable envelopes (by agent_id, action_type, scope)
      │
      ▼
Evaluate constitutional policies
  → DENY on any violation (short-circuit)
      │
      ▼
Evaluate operational policies
  → Compute ALLOW / MODIFY / ESCALATE
      │
      ▼
Evaluate safety policies
  → Attach ADVISORY signals
      │
      ▼
Compute final Decision (most-restrictive-wins)
      │
      ▼
Generate reasoning trace
      │
      ▼
Emit signed RPX Record
      │
      ▼
Return Decision to Gateway
```

#### 5.1.3 Safety Invariants

Kernels MUST enforce all four invariants unconditionally:

- **[K-INV-1]** A high-impact action MUST NOT be executed without a preceding `ALLOW` or `MODIFY` Decision.
- **[K-INV-2]** Every permission evaluation MUST produce an RPX Record.
- **[K-INV-3]** The Kernel MUST fail safe: if policy cannot be loaded or evaluated, the Decision MUST be `DENY`.
- **[K-INV-4]** Full request context MUST be preserved and recorded throughout the evaluation pipeline.

---

### 5.2 SSI Gateway

The Gateway is the protocol entry point for Agent requests.

#### 5.2.1 Responsibilities

- Accept requests from Agents in any supported protocol format
- Normalize requests to the SSI internal schema
- Route normalized requests to one or more Kernel instances
- Return Decisions and RPX record references to the requesting Agent
- Collect observability data (throughput, latency, error rates)

#### 5.2.2 Supported Transport Protocols

Conformant Gateways MUST support HTTP/REST. Support for gRPC, WebSocket, and GraphQL is OPTIONAL.

#### 5.2.3 Deployment Patterns

- **Single-Kernel** — One Kernel instance behind the Gateway. Suitable for development and low-volume deployments.
- **Multi-Kernel** — Multiple Kernel instances for redundancy and load distribution.
- **Federated** — Separate Gateways per policy domain, each backed by domain-specific Kernels.

---

## 6. Policy Evaluation Model

### Conflict Resolution

When multiple envelopes apply to a single request:

1. Evaluate all applicable envelopes independently.
2. Collect all Decisions.
3. Apply precedence: `DENY` > `MODIFY` > `ADVISORY` > `ALLOW`.
4. The highest-precedence Decision is the final Decision.

### Deny-by-Default

Any action not explicitly permitted by at least one applicable envelope MUST produce a `DENY` Decision. The absence of a matching policy is not permission.

### Escalation

When an `ESCALATE` action is triggered by an operational policy, the Kernel MUST:

1. Return `DENY` to the Agent immediately (action is blocked pending human review).
2. Write an RPX Record with `decision: "DENY"` and `reasoning` indicating escalation.
3. Route the request to the escalation target queue defined in the envelope.

Human approval via the escalation queue MAY produce a subsequent `ALLOW` Decision, which MUST be recorded as a new RPX Record.

---

## 7. Security Model

### Cryptographic Requirements

- All Governance Envelopes MUST be signed with Ed25519 or an equivalent algorithm offering at least 128-bit security.
- All RPX Records MUST be signed by the Kernel that produced them.
- Signature verification MUST occur before any envelope is applied to a request.

### Threat Mitigations

| Threat | Mitigation |
|--------|-----------|
| Policy tampering | Envelope signatures; Kernel rejects unsigned/invalid envelopes |
| Audit trail manipulation | Append-only RPX store; record-level signatures |
| Policy bypass | K-INV-1: no execution without prior Decision |
| Replay attacks | `record_id` (UUID v4) + `timestamp` in every RPX Record |
| Privilege escalation | Deny-by-default; most-restrictive-wins resolution |

For the full threat model, see [`/docs/safety/threat-model.md`](/docs/safety/threat-model.md).

---

## 8. Conformance

A system is **SSI Protocol v1.0 Conformant** if and only if it satisfies all MUST requirements in this specification.

### Conformance Levels

#### Level 1 — Core Protocol

The minimum viable conformant implementation.

- Implements the Gateway interface (HTTP/REST)
- Routes requests to a Kernel
- Kernel evaluates requests against at least one Governance Envelope
- Kernel emits signed RPX Records
- RPX Records are persisted to an append-only store
- Deny-by-default is enforced
- All four Kernel safety invariants (K-INV-1 through K-INV-4) are enforced

**Suitable for:** Development, research, proof-of-concept

#### Level 2 — Production Ready

Level 1 plus:

- Multi-layer envelope evaluation (at least two envelope scopes)
- Most-restrictive-wins conflict resolution
- Full reasoning trace in every RPX Record
- Human escalation queue for `ESCALATE` policies
- Emergency intervention mechanism (ability to halt all agent actions)
- Audit logs retained and accessible to authorized auditors

**Suitable for:** Production deployments, regulated industries

#### Level 3 — Mission Critical (Forthcoming)

Level 2 plus formal verification, redundant Kernel deployment, and third-party certification. Specification to be defined in SSI v1.1.

**Suitable for:** Financial, healthcare, critical infrastructure

### Conformance Claims

Implementations MAY claim conformance using the following form:

> `[Product Name] implements SSI Protocol v1.0, Level [1|2].`

Implementations MUST NOT claim a conformance level they do not satisfy. Deviations from SHOULD requirements MUST be documented with rationale.

For the full self-assessment checklist, see [`/docs/CONFORMANCE.md`](/docs/CONFORMANCE.md).

---

## 9. Versioning

The SSI Protocol uses semantic versioning.

| Version series | Meaning |
|----------------|---------|
| `v1.x.y` | Backward-compatible. Implementations targeting v1.0 remain conformant. |
| `v2.x.y` | Breaking changes. New conformance assessment required. |

Sub-component specs (RPX, Envelope, Kernel, Gateway) are versioned independently but MUST maintain compatibility within a major Protocol version.

---

## 10. References

- [Protocol Overview](/docs/overview.md)
- [RPX Format Specification](/docs/protocol/rpx.md)
- [Governance Envelope Specification](/docs/protocol/governance-envelopes.md)
- [Kernel Architecture](/docs/protocol/kernel.md)
- [Gateway Architecture](/docs/protocol/gateway.md)
- [Safety Architecture](/docs/safety/safety-architecture.md)
- [Threat Model](/docs/safety/threat-model.md)
- [Conformance Guide](/docs/CONFORMANCE.md)
- [Governance Model](/docs/governance/overview.md)
- [RFC 2119 — Key words for use in RFCs](https://www.rfc-editor.org/rfc/rfc2119)

---

*SSI Protocol v1.0. Published by the DeAlgo Foundation under CC BY 4.0.*
