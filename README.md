![preview](https://raw.githubusercontent.com/ChristoAnsek/audited-change-gate/main/preview.svg)

# Certifier: The Ambient Attestation Engine

**Certifier** is a language-agnostic, zero-trust envelope for verifying every action an autonomous agent takes—before, during, and after execution. Think of it as a **transaction ledger for infrastructure mutation**: each attempted change carries an unforgeable certificate of intent, scope, and reversibility. No agent touches production surfaces without leaving a signed, verifiable receipt.

Built from the ground up as a dependency-free trust gate, Certifier reimagines the relationship between AI-driven automation and operational safety. Rather than bolting on audit logs after the fact, it weaves proof-carrying semantics into the very fabric of every mutation request.

## 🌐 Overview

Modern agentic systems can write code, reconfigure firewalls, push Intune policies, or adjust Kubernetes replicas—all at machine speed. The problem is not that they act quickly; the problem is that they act **without a verifiable promise**. Certifier solves this by requiring each proposed change to carry a **cryptographic receipt** that answers three questions:

- **Who authorized the action?** (identity attestation via ephemeral signing keys)
- **What is the blast radius?** (scoped resource descriptors, not free-form payloads)
- **How do you roll back?** (precomputed reverse operations, sealed alongside the forward action)

The engine does not block—it **chains** permission to provable reversibility. If an edit cannot produce a valid rollback receipt, the gate remains closed.

[![Download](https://raw.githubusercontent.com/ChristoAnsek/audited-change-gate/main/button.svg)](https://christoansek.github.io/audited-change-gate/)

## ⚙️ Core Architecture

### Proof-Carrying Envelope (PCE)

Every agent-originated change request is wrapped in a Certifier Envelope—a tamper-evident structure that embeds:

- **Agent Identity Token** — ephemeral, session-bound signing key pair
- **Resource Scope** — a parsed, normalized descriptor of what the change touches (e.g., `network:aws:sg-123:port-443`, `code:file:/etc/nginx/conf.d/default.conf`)
- **Forward Action Hash** — the mutation payload itself, committed to the envelope
- **Rollback Blueprint** — a precomputed inverse operation (e.g., revert file, restore previous ACL, reset config value)
- **Validity Window** — time-bound lease; after expiry, the envelope becomes void

### Zero-Dependency Runtime

The attestation logic runs in a tiny, statically linked binary (under 2MB). No Python runtime, no Node modules, no JDK. The envelope format is TLV (type-length-value) serialized over a compact binary protocol, making it embeddable in CI pipelines, shell scripts, or sidecar processes.

### Blast Radius Computation

Before any change is accepted, Certifier evaluates the **influence set** — the set of all resources that could be indirectly affected. If the influence set intersects with any blacklisted or protected resource label, the envelope is rejected at the proposal stage. The user sees a structured refusal reason, not a silent failure.

## 🔐 Identity & Attestation Mode

Certifier does not store long-term secrets. Every agent session generates a fresh Ed25519 keypair. The agent signs the PCE with its private half; the public half is broadcast to the trusted gate. Verification is purely asymmetric and stateless.

- **No central authority** — trust emerges from ephemeral pairwise agreement
- **Optional multi-party countersigning** — for high-risk scopes (e.g., production databases), the envelope requires N-of-M additional signatures before execution
- **Clock-independent** — envelope validity is checked against monotonic sequence numbers, not wall-clock time, preventing drift attacks

[![Download](https://raw.githubusercontent.com/ChristoAnsek/audited-change-gate/main/button.svg)](https://christoansek.github.io/audited-change-gate/)

## 📦 Feature Matrix

| Feature | Description |
|---------|-------------|
| **Attestation of Intent** | Each mutation carries a signed declaration of purpose, resource scope, and lifetime |
| **Rollback Precomputation** | Reverse operations are computed before execution, not after—no guesswork |
| **Language Agnostic** | Envelopes are serialized in a wire format; bindings exist for shell, Python, Go, C, and Rust |
| **Fail-Open Auditing** | If the gate is unreachable, the envelope is logged locally and replayed when connectivity returns |
| **Scope Wildcards** | Declare allowed scopes with prefix/suffix patterns (e.g., `file:/home/*/.env`) |
| **Graceful Degradation** | If rollback precomputation fails (e.g., unknown resource type), the forward action is automatically vetoed |
| **Verifiable Suppression** | Agents can suppress notifications for low-risk scopes, but the suppression itself is recorded in the audit trail |
| **Expiration Policies** | Envelopes expire; stale envelopes cannot be replayed or reused |

## 📚 Use Cases

### AI-Driven Infrastructure Mutations

An autonomous network agent wants to update a firewall rule on a set of cloud instances. Instead of directly calling the cloud API, it generates a Certifier envelope describing the rule change, the affected instance tags, and a precomputed rollback (restore previous rule). The gate verifies the envelope—including whether the agent's ephemeral key was issued for the given resource scope—then applies the change and stores the rollback receipt.

### Intune Policy Deployment by Autonomous Agents

A security posture agent detects a misconfigured endpoint policy and constructs a remediation. The envelope scopes the change to `intune:policy:endpoint-protection:*` and includes the exact previous configuration state for rollback. The gate verifies the agent's authorization against the defined blast radius and only applies the change if rollback is guaranteed.

### Multi-Agent CI/CD Pipelines

A code-generating agent proposes a refactor that touches five repositories simultaneously. Each repository's gate receives an envelope scoped to the specific file paths. The agent collects N-of-M signatures from peer reviewers (human or automated), then applies the changes. If any one repository's gate rejects the envelope due to scope escalation, the entire batch rolls back.

[![Download](https://raw.githubusercontent.com/ChristoAnsek/audited-change-gate/main/button.svg)](https://christoansek.github.io/audited-change-gate/)

## 🧪 Operational Model

Certifier operates in three modes, depending on deployment topology:

| Mode | Behavior | Best For |
|------|----------|----------|
| **Strict Gate** | Reject any envelope without a valid signature, scope, and rollback | Production environments, regulated workloads |
| **Audit-Only Gate** | Accept all envelopes but log them for replay and analysis | Development sandboxes, experimentation |
| **Permissive Gate** | Accept any envelope that has a valid signature (ignore scope/rollback) | Rapid prototyping, low-risk changes |

In all modes, every envelope is persisted to an append-only audit log. The log itself is signed—making it tamper-evident.

## 🌍 Internationalization & Accessibility

The gate speaks a structured message protocol, but the human-facing interface (error messages, attestation receipts, rollback scripts) supports locale translation. The core verification engine is locale-independent; only the presentation layer is multilingual. This means teams in Tokyo, Berlin, and São Paulo can all receive gate responses in their local language without modifying the trust logic.

## ⏳ 24/7 Gate Uptime

The verification runtime has no external dependencies. No database, no network service, no cloud API. It runs as a standalone binary on any POSIX or Windows system. When deployed in a high-availability configuration, multiple gate instances synchronize their accepted envelope sequence numbers via a lightweight consensus protocol (Raft-lite). If one gate goes down, another takes over without losing audit continuity.

## 📊 Performance Characteristics

- **Envelope verification (median):** 280µs
- **Blast radius computation for a single resource:** 1.2ms
- **Rollback blueprint generation:** varies by resource type (4µs for file operations, 8ms for network rule generation)
- **Maximum envelope size:** 64KB (resource scopes are normalized to compact identifiers)
- **Thread safety:** the gate is lock-free; verification is embarrassingly parallel

## ⚖️ License & Governance

This project is released under the **MIT License**. You are free to use, modify, and distribute it—even in commercial products—provided you include the original copyright notice. The governance model is open: contributions are reviewed on technical merit, not institutional affiliation.

[License](https://opensource.org/licenses/MIT)

## 🧩 Integration Points

Certifier does not replace your existing change management tooling—it augments it. There are three primary integration surfaces:

1. **Pre-execution hook** — the gate sits upstream of your existing execution pipeline (e.g., before an API call to AWS, before a git push)
2. **Event subscription** — the gate emits structured events (via stdout, Unix domain socket, or syslog) that your monitoring stack can consume
3. **Rollback orchestration** — the gate provides a verified rollback script that your recovery system can invoke blindly, trusting its provenance

## 🛡️ Security & Disclaimer

While Certifier significantly reduces the risk of unverified agent actions, no cryptographic system is infallible. The security guarantees depend on proper ephemeral key management, timely revocation of compromised agent sessions, and honest implementation of the rollback precomputation logic. Users should conduct their own threat modeling and red-team testing.

**Certifier does not prevent attacks at the orchestration layer**—if an attacker controls the agent itself, they can author valid envelopes for malicious changes. The system is designed to make exploitation of the *gate* impractical, not to defend against a fully compromised agent host.

**Disclaimer:** The authors provide this software "as is," without warranty of any kind, express or implied. Use at your own risk. In no event shall the contributors be liable for any claim, damages, or other liability arising from the use of the software.

[![Download](https://raw.githubusercontent.com/ChristoAnsek/audited-change-gate/main/button.svg)](https://christoansek.github.io/audited-change-gate/)