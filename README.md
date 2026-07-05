# authgate-specs

Formal RFC specifications for the [authgate-kernel](https://github.com/Aliipou/authgate-kernel) capability-governance runtime.

## RFCs

| RFC | Title | Status |
|-----|-------|--------|
| [RFC-001](rfc/RFC-001-capability-semantics.md) | Capability Semantics | Draft |
| [RFC-002](rfc/RFC-002-delegation-algebra.md) | Delegation Algebra | Draft |
| [RFC-003](rfc/RFC-003-revocation-model.md) | Revocation Model | Draft |
| [RFC-004](rfc/RFC-004-distributed-authority-resolution.md) | Distributed Authority Resolution | Draft |
| [RFC-005](rfc/RFC-005-capability-attenuation.md) | Capability Attenuation | Draft |
| [RFC-006](rfc/RFC-006-trust-root-semantics.md) | Trust Root Semantics | Draft |

## Relationship to authgate-kernel

These specs define the formal semantics that `authgate-kernel` implements. The kernel enforces the invariants stated here. When a spec and the implementation diverge, the spec is authoritative.

## Where authority sits in the system

The full system is **Legitimacy ⊥ Authority**, both layers being one theory made
executable — the authority layer these RFCs specify is *not* neutral plumbing.
Canonical pipeline (locked): identity admission → FDK legitimacy (DENY-only) →
**authority (grant within legitimacy — specified here)** → PEP execute + audit.
**Invariant:** legitimacy may only DENY; authority never overrides a legitimacy
denial. These RFCs formalize only the authority half; legitimacy is specified in the
FDK line.

## Status

All RFCs are currently in **Draft** status. They describe the intended semantics, not a finalized standard.

## License

**Source-available** under the [PolyForm Noncommercial License 1.0.0](LICENSE) — see also [`NOTICE`](NOTICE).
Research, educational, and evaluation use are allowed with attribution; commercial use, production
deployment, resale, and SaaS require a separate commercial license. Contact **Ali Pourrahim —
Alipourrahim.ap@gmail.com**.
