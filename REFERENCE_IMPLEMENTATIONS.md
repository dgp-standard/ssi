# SSI Protocol - Reference Implementations

SSI Protocol maintainers provide **maintained reference implementations** to demonstrate compliance patterns.

**This file is non-normative.** Compliance requirements are defined in the constitutional documents.

---

## SSI Kernel

Decision evaluation engine with policy-based authorization.

**Location:** [`reference/kernel/`](reference/kernel/)

**Features:**
- Policy-based decision evaluation
- Fail-closed error handling
- RPX record generation
- Operator escalation workflows

**Documentation:** See [IMPLEMENTATION.md](IMPLEMENTATION.md)

---

## SSI Gateway

RPX record management, hash chain verification, and audit interfaces.

**Location:** [`reference/gateway/`](reference/gateway/)

**Features:**
- RPX record storage (PostgreSQL or in-memory)
- Hash chain integrity verification
- REST API for decision requests
- Audit log export

**Documentation:** See [IMPLEMENTATION.md](IMPLEMENTATION.md)

---

## Example Governance Implementation

The protocol authors maintain a reference governance implementation that demonstrates advanced patterns including:
- Human authority escalation workflows
- Cryptographic linkage to SSI audit trails
- Multi-layer policy evaluation

**Note:** Advanced governance implementations build on SSI Protocol but are not part of the constitutional standard. They demonstrate how sophisticated governance can be implemented while maintaining SSI compliance.

---

## SDKs

### TypeScript/JavaScript
**Location:** [`sdks/typescript/`](sdks/typescript/)  
**Status:** In development

### Python
**Status:** Planned for v1.1.0

### Rust
**Status:** Planned for v2.0.0

---

## Community Implementations

Third-party SSI-compliant implementations can be listed on the [Community Implementations](https://github.com/dgp-standard/ssi-protocol-oss/wiki/Community-Implementations) wiki page (to be established).

To register an implementation:
1. Pass Level 1 compliance tests
2. Submit documentation showing conformance to constitutional requirements
3. Include link to public verification report (if available)

---

**Reference implementations demonstrate the protocol. Compliance is defined in the constitutional documents.**
