# Red-Team Findings — Decision OS authorization invariants

Falsification-first red-team (read-only against source; PoCs in
`…/d24ac95a-…/scratchpad/redteam/`). Ranked. Crypto core held; all breaks are in
**statefulness/durability** — the failures that only appear under real deployment
(multiple replicas, attacker with log-write access).

## HARD BREAKS

### HB-1 — "One-time" token is not one-time across Executor instances (replay/double-spend)
- **Where:** decision-os-min; control-plane + decision-kernel-core (same flaw)
- **Invariant:** #6 (token one-time)
- **PoC:** `poc1_replay_cross_instance.py`, `poc9_controlplane_replay_and_nonce.py`
- **What:** `_spent` is an in-memory set per instance. Same-instance replay is
  correctly rejected; a second worker/replica/restart has an empty spent-set and
  re-spends the token within its 30s TTL → **one decision, two effects**.
- **Fix:** shared, durable, atomic spent-store (DB unique constraint on token_id,
  or Redis SETNX); fail closed if unreachable.

### HB-2 — Audit tail-truncation is undetectable
- **Where:** decision-os-min `HashLog`
- **Invariant:** #5 (tamper-evident chain)
- **PoC:** `poc3_audit_truncation.py`
- **What:** delete the last N entries → `verify()` still True; new entries chain
  onto the truncated head. No length/head commitment. NOTE: `audit-ledger`'s
  `HashChainedAudit` already fixes this (head()/anchor/verify_against_anchor) —
  the distilled `HashLog` **regressed** that hardening.
- **Fix:** port the anchor/head mechanism from audit-ledger into HashLog.

### HB-3 — PEP executes effects with zero audit entries
- **Where:** decision-os-min
- **Invariant:** #5 (exactly one audit entry per routed decision)
- **PoC:** `poc5_effect_without_audit.py`
- **What:** `Kernel` and `Executor` are public exports; used directly they run the
  effect and write no audit — audit lives only in `DecisionOS.handle()`, not in
  the enforcement point. Audit is by-placement, not enforced.
- **Fix:** require an audit sink in `Executor` and write inside `execute()`, or
  force all callers through `handle()`.

## WEAKNESSES

- **W-1 — nonce/action_ref not part of the action binding** (all three repos;
  invariant #7). `action_fingerprint` ignores nonce/action_ref; combined with HB-1
  a captured ALLOW can authorize an attacker-constructed action object. The
  integration invariant suite is blind to this (re-derives the fingerprint the
  same way). Fix: fold nonce/action_ref into the binding and check
  action.nonce == decision.action_ref in the executor.
- **W-2 — `default=str` canonicalization collision + mutate-after-auth**
  (decision-os-min, decision-kernel-core; invariant #7). An object stringifying to
  "100" collides with the string "100"; a payload object mutated after hashing runs
  the mutated value. Requires attacker-controlled object types. Fix: reject
  non-JSON-primitive payload values before fingerprinting (strict encoder).
- **W-3 — audit does not reflect execution outcome or payload** (decision-os-min).
  Records pre-execution verdict only; ALLOW-but-refused looks identical to
  ALLOW-and-ran; $1 and $1M produce identical lines. Fix: record after execution
  with executed/refused flag + payload digest.

## Invariants that HELD (tried hard, could not break)
#1 forged/tampered signed decision rejected · #2 no token forgery without the
private key · #3 advisory can only tighten, never loosen a DENY · #4 PEP never
constructs a decision · #6 same-instance replay rejected · #7 primitive int-vs-str
distinct; tool-swap runs the authorized capability, not the action's tool field ·
#8 legitimacy denial is final; CONTAIN empty allowlist refuses (no silent allow).

## Bottom line
The cryptographic core is sound. The dangerous breaks (HB-1 cross-instance
double-spend, HB-2 silent audit truncation) are exactly what in-repo unit suites
miss because they need a multi-replica / log-write-access deployment to appear.
Fix priority: HB-1, HB-2, HB-3, then W-1, W-2, W-3.
