# RFC-007: Capability Expiration and TTL Semantics

**Status:** Draft  
**Phase:** 6 (stabilization)  
**Depends on:** RFC-001 (capability semantics), RFC-003 (revocation model)

---

## Abstract

This RFC defines the formal semantics for capability expiration (TTL) in the
authgate-kernel registry. It specifies the clock model, expiry enforcement points,
and interaction with cascading revocation.

---

## Motivation

Claims in the current implementation carry an optional `expires_at: float | None`
field (Unix timestamp). Expiry is checked in `is_valid()` but is not enforced
proactively — claims remain in the registry until `expire_stale()` is called.

This creates a gap: between expiry and the next `expire_stale()` call, a claim
that `is_valid()` returns False for will still appear in the index. This is
correct behavior (it is filtered at lookup time), but the semantics need to be
formally specified.

---

## Formal Semantics

**Definition (Valid claim at time T):**
```
is_valid(c, T) ⟺ c.confidence > 0 ∧ (c.expires_at = None ∨ T < c.expires_at)
```

**Enforcement guarantee:**
For any call to `verify(action)` at system time T, all claim lookups use
`is_valid(c, T)`. Expired claims are filtered and cannot contribute to a PERMITTED result.

**Proactive cleanup:**
`expire_stale()` removes expired claims from the registry and updates the index.
This is optional and affects storage efficiency, not security correctness.

---

## Session-Scoped Capabilities

A session-scoped capability is a claim with `expires_at = session_end_time`.
The session framework (outside TCB) is responsible for:
1. Computing session_end_time
2. Creating claims with the correct expiry
3. Calling `expire_stale()` or `revoke_on_resource()` at session end

---

## TTL vs Cascading Revocation Interaction

When a delegator's claim expires:
- Claims delegated by that entity remain valid until their own `expires_at`
- Expiry is NOT cascading by default — each claim expires independently
- To enforce cascading expiry, use `revoke_cascading()` explicitly at session end

This differs from `revoke_cascading()` (which propagates immediately) and from
pure TTL (which relies on time-based expiry per claim).

---

## Open Questions

- Should expiry cascade by default? (Current: No — requires explicit revocation)
- What clock synchronization is required for multi-node deployments?
- Should `expires_at` be a duration (relative) or timestamp (absolute)?
