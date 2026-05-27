# RFC-002: Delegation Algebra

**Status:** Draft  
**Category:** Core  
**Depends on:** RFC-001 (Capability Semantics)  

---

## Abstract

Defines the algebraic structure governing how capabilities may be delegated between agents in the freedom-kernel runtime. Authority is modeled as a directed acyclic graph (DAG) with bounded depth, and delegation is governed by a set of formal rules that preserve the attenuation invariant established in RFC-001.

---

## 1. Delegation Graph

### 1.1 Definition

Let `A` be the set of all agents registered in the system, and let `P ⊂ A` be the set of human principals (trust roots). A **delegation graph** `G = (V, E)` is defined as:

- `V ⊆ A`: the set of agents participating in at least one delegation chain
- `E ⊆ V × V × Cap × AuthorityBudget`: directed edges, where each edge `(u, v, C, B)` represents a delegation from agent `u` to agent `v` of capability set `C` under authority budget `B`

### 1.2 DAG Constraints

The delegation graph MUST satisfy the following structural invariants at all times:

**Constraint 1 — Acyclicity:**
```
∀ path (v₀, v₁, ..., vₙ) in G: v₀ ≠ vₙ
```
No agent may directly or transitively delegate to itself or to any agent that is an ancestor in its own delegation chain. A delegation that would introduce a cycle MUST be rejected with `CYCLE_DETECTED`.

**Constraint 2 — Bounded Depth:**
```
MAX_DELEGATION_DEPTH = 16

∀ path p from root to leaf in G: |p| ≤ MAX_DELEGATION_DEPTH
```
The length of any delegation chain from a human principal to a leaf agent MUST NOT exceed `MAX_DELEGATION_DEPTH`. A delegation request that would produce a path exceeding this bound MUST be rejected with `DEPTH_EXCEEDED`.

**Rationale:** The depth bound limits the surface area of cascading revocation, reduces the cost of chain validation, and constrains the blast radius of a compromised intermediate agent.

**Constraint 3 — Principal Anchoring:**
```
∀v ∈ V: ∃p ∈ P, ∃ directed path from p to v in G
```
Every agent in the delegation graph MUST have at least one human principal as an ancestor. Orphaned authority — authority that cannot be traced back to a human principal — is invalid and MUST NOT be exercised.

---

## 2. Authority Budget

Each delegation edge `(u, v, C, B)` carries an **authority budget** `B`, a structured constraint on the scope of delegated authority. The authority budget is defined as:

```
AuthorityBudget = {
  max_actions:          Option<u64>,   // max total capability invocations
  max_child_agents:     Option<u32>,   // max agents that v may spawn
  max_delegation_depth: Option<u8>,    // max further depth v may delegate to
  expires_at:           Option<Timestamp>,  // absolute expiry time
  revocable:            bool,          // whether delegator may revoke
}
```

### 2.1 Budget Attenuation

An authority budget `B'` granted by `v` to a child `w` MUST satisfy:

```
B'.max_actions          ≤ B.max_actions          (if both Some)
B'.max_child_agents     ≤ B.max_child_agents      (if both Some)
B'.max_delegation_depth ≤ B.max_delegation_depth − 1  (depth decrements)
B'.expires_at           ≤ B.expires_at            (cannot extend lease)
B'.revocable            ∨ ¬B.revocable            (can only add revocability, not remove)
```

A child agent MUST NOT issue a budget `B'` that exceeds its own received budget `B` in any dimension.

### 2.2 Budget Exhaustion

When `max_actions` reaches zero, the capability grant is exhausted and MUST be treated as revoked. The runtime MUST enforce this atomically using a decrement-and-compare operation to prevent race conditions.

---

## 3. Formal Delegation Rules

### Rule D1 — Attenuation Rule

For any delegation `(u → v, C_v, B_v)` where `(p → ... → u, C_u, B_u)` is the path from a principal to `u`:

```
C_v ⊆ C_u
```

Agent `v` may not receive any capability kind or resource scope that `u` does not itself hold. Delegation MUST be rejected with `ATTENUATION_VIOLATION` otherwise.

**Formal statement:** Let `holds(a)` denote the capability set of agent `a`. Then:

```
holds(v) = ⋂ { C_e | e is an incoming edge to v }
```

When multiple delegation paths reach the same agent, the agent holds only the intersection of all incoming grants (conservative semantics). See Section 5 for intersection details.

### Rule D2 — Transitivity Rule

Delegation is transitively valid if and only if every edge in the chain satisfies D1:

```
holds(vₙ) ⊆ holds(vₙ₋₁) ⊆ ... ⊆ holds(v₀)
```

The runtime MUST NOT short-circuit chain validation. Each link in the chain must be individually verified. A valid root grant does not imply validity of downstream edges if an intermediate edge was forged or improperly attenuated.

**Corollary:** An agent's effective capability set is bounded by the weakest grant in any path from a principal to that agent. This ensures that a compromised intermediate node cannot amplify authority for downstream agents.

### Rule D3 — Revocation Rule

Revocation is monotone-increasing and never undone within a session:

```
revoked(e, t₁) ∧ t₂ > t₁  ⟹  revoked(e, t₂)
```

Once a delegation edge is revoked, it cannot be un-revoked. A new delegation may be issued, but it is a distinct edge with a new identifier and timestamp. See RFC-003 for full revocation semantics.

### Rule D4 — Non-Self-Grant Rule

```
∀(u → v, C, B) ∈ G: u ≠ v
```

An agent cannot delegate to itself. Self-delegation is rejected with `SELF_DELEGATION_PROHIBITED`.

### Rule D5 — Principal Non-Delegation of Catastrophic Capabilities

```
∀(p → v, C, B) where p is a machine agent: POLICY_MODIFY ∉ C ∧ SYSTEM_PROMPT_EDIT ∉ C
```

Capabilities classified as `Catastrophic` (see RFC-001, §1.2) MUST NOT be delegated by machine agents. They may only be granted directly by a registered human principal acting via an authenticated out-of-band channel.

---

## 4. Delegation Chain Validation Algorithm

The following algorithm is used by the runtime to validate a capability exercise request from agent `v`:

```
function validate_chain(v: Agent, cap: Capability, resource: Resource) -> Result<(), ValidationError>:
    chain = compute_shortest_delegation_path(trust_root, v)
    
    if chain is None:
        return Err(NO_VALID_CHAIN)
    
    if |chain| > MAX_DELEGATION_DEPTH:
        return Err(DEPTH_EXCEEDED)
    
    for each edge e = (u, w, C, B) in chain:
        if revoked(e):
            return Err(EDGE_REVOKED(e.id))
        
        if expired(B.expires_at):
            return Err(EDGE_EXPIRED(e.id))
        
        if cap ∉ C:
            return Err(CAPABILITY_NOT_IN_GRANT(e.id, cap))
        
        if resource ∉ e.resource_scope:
            return Err(RESOURCE_OUT_OF_SCOPE(e.id, resource))
        
        if B.max_actions == Some(0):
            return Err(BUDGET_EXHAUSTED(e.id))
    
    // Decrement action counters atomically
    for each edge e in chain:
        if e.B.max_actions is Some(n):
            atomic_decrement(e.B.max_actions)
    
    return Ok(())
```

**Note:** Implementations MUST use the shortest valid path for validation to ensure the most restrictive set of constraints is applied. When multiple paths exist (diamond delegation patterns), all paths MUST satisfy the invariants.

---

## 5. Claim Intersection Semantics

When agent `v` receives delegations from multiple sources `u₁, u₂, ..., uₙ` (a diamond pattern in the DAG), its effective capability set is the intersection of all incoming grants:

```
holds(v) = C₁ ∩ C₂ ∩ ... ∩ Cₙ
```

This is the **conservative intersection rule**. It prevents an agent from accumulating authority beyond what any single delegation path would permit.

**Rationale:** An alternative rule (union semantics) would allow an agent to accumulate capabilities by soliciting multiple delegations. The intersection rule eliminates this attack surface while permitting valid multi-principal co-authorization patterns (where all principals must agree).

### 5.1 Budget Intersection

When multiple incoming budgets apply, the effective budget is also the intersection (minimum) of all budget dimensions:

```
effective_budget.max_actions          = min(B₁.max_actions, ..., Bₙ.max_actions)
effective_budget.max_child_agents     = min(B₁.max_child_agents, ..., Bₙ.max_child_agents)
effective_budget.max_delegation_depth = min(B₁.max_delegation_depth, ..., Bₙ.max_delegation_depth)
effective_budget.expires_at           = min(B₁.expires_at, ..., Bₙ.expires_at)
effective_budget.revocable            = B₁.revocable ∨ B₂.revocable ∨ ... ∨ Bₙ.revocable
```

The `revocable` field uses disjunction: if any principal can revoke, the delegation is revocable.

---

## 6. Normative Requirements

- MUST: The delegation graph MUST remain acyclic at all times. Cycle detection MUST be performed before any delegation edge is committed.
- MUST: Delegation chains MUST NOT exceed `MAX_DELEGATION_DEPTH = 16`.
- MUST: Every agent in the delegation graph MUST have a traceable path to a registered human principal.
- MUST: The Attenuation Rule (D1) MUST be enforced atomically at delegation time, not lazily at use time.
- MUST: `POLICY_MODIFY` and `SYSTEM_PROMPT_EDIT` MUST NOT appear in any machine-issued delegation edge.
- SHOULD: Implementations SHOULD store the delegation graph in a persistent, append-only structure to enable audit and forensic reconstruction.
- MUST NOT: Budget fields MUST NOT be mutated after issuance except by decrement of `max_actions` during use, or by full revocation.
