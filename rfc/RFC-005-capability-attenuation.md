# RFC-005: Capability Attenuation

**Status:** Draft  
**Category:** Core  
**Depends on:** RFC-001 (Capability Semantics), RFC-002 (Delegation Algebra)  

---

## Abstract

Defines the formal mathematical structure of capability attenuation in the freedom-kernel runtime. Attenuation is the mechanism by which delegated authority is strictly weakened relative to the delegating agent's authority. This RFC establishes attenuation as a partial order on capability sets, proves that the capability system forms a lattice under this order, analyzes the compositional properties of chained attenuation, and extends the model to confidence and time dimensions. A proof that attenuation prevents privilege escalation is provided in the final section.

---

## 1. Formal Definition of Attenuation

### 1.1 Capability Sets

Let `K` be the finite, closed set of capability kinds defined in RFC-001 §1.1. Let `R` be the set of all resources in the system. A **capability grant** is a pair `(k, r) ∈ K × R`, representing authority to perform operation `k` on resource `r`.

Let `Cap = 𝒫(K × R)` be the power set of all possible capability grants. An agent's authority is a set `C ∈ Cap`.

### 1.2 Attenuation as a Binary Relation

**Definition (Attenuation):** A capability set `C'` is an **attenuation** of `C`, written `C' ⊑ C`, if and only if:

```
C' ⊑ C  ⟺  C' ⊆ C
```

That is, attenuation is set containment. An attenuated capability set contains no capability grants beyond those in the original set.

This definition captures the intuition: you can restrict what you give, but you cannot give more than you have.

---

## 2. Lattice Structure

### 2.1 Partial Order

The relation `⊑` is a partial order on `Cap`. We verify the three properties:

**Reflexivity:** `C ⊑ C` because `C ⊆ C`. An agent's capability set is always an attenuation of itself (trivial, non-strict attenuation).

**Antisymmetry:** If `C₁ ⊑ C₂` and `C₂ ⊑ C₁`, then `C₁ ⊆ C₂` and `C₂ ⊆ C₁`, which implies `C₁ = C₂`. Two mutually attenuating capability sets are identical.

**Transitivity:** If `C₁ ⊑ C₂` and `C₂ ⊑ C₃`, then `C₁ ⊆ C₂ ⊆ C₃`, so `C₁ ⊆ C₃`, i.e., `C₁ ⊑ C₃`. This is the mathematical foundation of RFC-002 Rule D2 (Transitivity Rule).

Therefore, `(Cap, ⊑)` is a partially ordered set (poset).

### 2.2 Bounded Lattice

`(Cap, ⊑)` is in fact a **bounded lattice** with:

- **Meet (greatest lower bound):** `C₁ ⊓ C₂ = C₁ ∩ C₂`
- **Join (least upper bound):** `C₁ ⊔ C₂ = C₁ ∪ C₂`
- **Bottom element (⊥):** `∅` — the empty capability set (no authority)
- **Top element (⊤):** `K × R` — all capabilities on all resources (maximal authority)

**Proof that this is a lattice:**

For any `C₁, C₂ ∈ Cap`:
- `C₁ ∩ C₂ ⊆ C₁` and `C₁ ∩ C₂ ⊆ C₂`, so `C₁ ∩ C₂ ⊑ C₁` and `C₁ ∩ C₂ ⊑ C₂` (lower bound).
- For any `D` such that `D ⊑ C₁` and `D ⊑ C₂`: `D ⊆ C₁` and `D ⊆ C₂`, so `D ⊆ C₁ ∩ C₂`, i.e., `D ⊑ C₁ ∩ C₂` (greatest lower bound). ✓

Dually for join. The bounded lattice `(Cap, ⊑, ∩, ∪, ∅, K×R)` is well-formed. ∎

### 2.3 Significance of the Lattice Structure

The lattice structure has direct operational significance:

- **Meet (intersection):** Used in RFC-002 §5 for multi-principal claim intersection. The effective authority of an agent receiving delegations from multiple principals is the meet of their grants.
- **Join (union):** Represents the maximal authority achievable by combining grants. The system MUST enforce that no agent's actual grant exceeds any individual delegation's authority (i.e., union semantics are explicitly prohibited for multi-principal claims).
- **Bottom (∅):** A revoked agent's effective authority collapses to ⊥.
- **Monotone-decreasing chains:** Delegation chains form strictly descending chains in the lattice (by the attenuation invariant), from the trust root's ⊤-adjacent grant down to the leaf agent's authority.

---

## 3. Compositional Attenuation

### 3.1 Chained Attenuation

Consider a delegation chain `p₀ → p₁ → ... → pₙ` with capability sets `C₀ ⊇ C₁ ⊇ ... ⊇ Cₙ`. The composed attenuation from `p₀` to `pₙ` is:

```
C_composed = C₀ ∩ C₁ ∩ ... ∩ Cₙ = Cₙ
```

Since the chain is monotonically decreasing, the terminal set `Cₙ` is already the intersection of all sets in the chain. The **effective attenuation** of a chain is simply the capability set of the leaf agent.

**Corollary:** Adding intermediate delegation steps never increases authority. Each additional hop can only maintain or further restrict the leaf agent's capability set. This is the formal basis for the safety of transitive delegation.

### 3.2 Attenuation Composition is Idempotent

If an agent attenuates its own grant and then re-attenuates (for example, by receiving the same capability through two different paths, both attenuated):

```
C' ⊑ C  and  C'' ⊑ C'  ⟹  C'' ⊑ C' ⊑ C
```

Re-attenuation is idempotent in the sense that the result is always a further restriction. There is no mechanism by which re-attenuation can recover authority.

### 3.3 Attenuation Under Restriction Functions

In practice, attenuation is often applied via a **restriction function** `f: Cap → Cap` that maps a capability set to a subset. For a restriction to be a valid attenuation:

```
∀C ∈ Cap: f(C) ⊆ C   (outputs are always subsets of inputs)
```

The composition of two restriction functions `f` and `g` is also a valid restriction:

```
(f ∘ g)(C) = f(g(C)) ⊆ g(C) ⊆ C  ✓
```

This closure property ensures that composed attenuations remain valid attenuations, enabling modular attenuation policies.

---

## 4. Confidence Attenuation

### 4.1 Confidence as a Capability Dimension

Beyond the binary question of whether a capability is granted, the freedom-kernel introduces a **confidence score** `κ ∈ [0, 1]` on each capability grant, representing the degree to which the issuing principal trusts the recipient to exercise the capability safely and correctly.

A capability grant with confidence is represented as a triple `(k, r, κ)`.

The **confidence-augmented capability set** is `C ⊆ K × R × [0, 1]`.

### 4.2 Confidence Attenuation Rule

**Definition:** A confidence-augmented capability set `C'` is a **confidence attenuation** of `C`, written `C' ⊑_κ C`, if:

```
∀(k, r, κ') ∈ C': ∃(k, r, κ) ∈ C such that κ' ≤ κ
```

A delegated grant MUST NOT have higher confidence than the delegating agent's own grant of the same capability.

**Rationale:** Confidence encodes epistemic trust, not just structural authority. A delegating agent cannot certify that a recipient is more trustworthy than the agent itself has been found to be. Confidence escalation through delegation would allow a chain of slightly-trusted agents to manufacture high-confidence authority.

### 4.3 Confidence Composition

For a delegation chain `p₀ → p₁ → ... → pₙ` with confidence values `κ₀ ≥ κ₁ ≥ ... ≥ κₙ`:

The effective confidence of `pₙ`'s grant is `κₙ`, which is bounded by the confidence at each link.

**Optional multiplicative model:** Implementations MAY use a multiplicative confidence model:

```
κ_effective = κ₀ × κ₁ × ... × κₙ
```

This models confidence degradation over chains: each delegation introduces a small reduction in confidence, reflecting the compounding epistemic uncertainty of long chains. Under this model, `κ_effective ≤ κᵢ` for all `i`, satisfying the attenuation rule.

---

## 5. Time Attenuation

### 5.1 Temporal Capability Grants

A time-attenuated capability grant carries a validity interval `[issued_at, expires_at]` (using the `expires_at` field of `AuthorityBudget` from RFC-002 §2). A grant is only valid for capability exercises occurring within this interval.

### 5.2 Time Attenuation Rule

**Definition:** A time-attenuated grant with interval `[t₀', t₁']` is a **time attenuation** of a grant with interval `[t₀, t₁]` if:

```
t₀ ≤ t₀'  and  t₁' ≤ t₁
```

The delegated lease must start no earlier and end no later than the parent lease. In practice, since the delegated grant is issued after the parent (at `t ≥ t₀`), the binding constraint is:

```
expires_at_child ≤ expires_at_parent
```

**This is the lease attenuation constraint of RFC-002 §5.1.**

### 5.3 Time Composition

For a chain of time-attenuated grants with intervals `[t₀ᵢ, t₁ᵢ]`, the effective validity interval of the leaf agent is:

```
effective_interval = [max(t₀₀, t₀₁, ..., t₀ₙ), min(t₁₀, t₁₁, ..., t₁ₙ)]
```

The effective window is the intersection of all intervals in the chain. This is equivalent to computing the meet of intervals in the lattice of intervals ordered by containment, which is consistent with the general attenuation framework.

---

## 6. Proof that Attenuation Prevents Privilege Escalation

### 6.1 Theorem (Privilege Escalation Impossibility)

**Theorem:** Under the freedom-kernel attenuation model, no agent can obtain a capability grant that is strictly greater (in the `⊑` order) than any grant held by a registered human principal in the trust root set `P`.

Formally:

```
∀v ∈ A, ∀p ∈ P: holds(v) ⊑ holds(p)
```

where `holds(a)` denotes the effective capability set of agent `a`, and `P` is the set of human principals whose grants authorize the delegation chain reaching `v`.

### 6.2 Proof

We proceed by induction on the length of the delegation chain from a human principal to agent `v`.

**Base case:** Agent `v = p` for some `p ∈ P`. Then `holds(v) = holds(p)`, and `holds(p) ⊑ holds(p)` holds trivially by reflexivity. ✓

**Inductive step:** Assume the theorem holds for all agents at depth `d`. Consider an agent `v` at depth `d+1`, which received its grant from some agent `u` at depth `d`.

By the inductive hypothesis: `holds(u) ⊑ holds(p)` for some `p ∈ P` on the path to `u`.

By the Attenuation Rule (RFC-002 Rule D1): `holds(v) ⊆ holds(u)`, i.e., `holds(v) ⊑ holds(u)`.

By transitivity of `⊑`: `holds(v) ⊑ holds(u) ⊑ holds(p)`, so `holds(v) ⊑ holds(p)`. ✓

**DAG case (multiple paths):** If `v` receives delegations from multiple agents `u₁, ..., uₙ`, by RFC-002 §5 (conservative intersection):

```
holds(v) = holds(u₁) ∩ holds(u₂) ∩ ... ∩ holds(uₙ)
```

Each `holds(uᵢ) ⊑ holds(pᵢ)` by the inductive hypothesis. Since `holds(v) ⊆ holds(uᵢ)` for all `i`, we have `holds(v) ⊑ holds(uᵢ) ⊑ holds(pᵢ)` for all `i`. ✓

**Conclusion:** By induction, no agent at any depth in the delegation graph holds a capability grant exceeding that of any human principal on the path to that agent. Privilege escalation through delegation is formally impossible under the attenuation model. ∎

### 6.3 Conditions for the Proof to Hold

The proof is conditional on the following invariants being enforced by the runtime:

1. **Attenuation Rule (RFC-002 D1) is enforced atomically at delegation time.** If the runtime permits delegation without checking this rule, escalation is possible.
2. **Conservative intersection is used for multi-path claims (RFC-002 §5).** If union semantics were used, an agent could accumulate authority beyond any single principal's grant.
3. **No self-grant is possible (RFC-001 §5, RFC-002 D4).** Self-grant would allow an agent to bootstrap authority from nothing.
4. **The key registry is not compromised.** If an attacker can register a malicious principal key and bootstrap a trust root, the proof does not apply to the subtree rooted at that key.
5. **The audit chain and delegation store are consistent.** Forged or unrecorded delegations could introduce edges that bypass the attenuation check.

---

## 7. Normative Requirements

- MUST: Every delegation MUST satisfy `C_child ⊆ C_parent` for capability sets. The runtime MUST reject delegations that violate this.
- MUST: Confidence scores, if used, MUST satisfy `κ_child ≤ κ_parent`. Confidence escalation MUST be rejected.
- MUST: Time intervals MUST satisfy `expires_at_child ≤ expires_at_parent`. Lease extension MUST be rejected.
- MUST: Multi-path grants MUST use intersection (meet) semantics. Union semantics for multi-source grants are PROHIBITED.
- SHOULD: Implementations SHOULD use the multiplicative confidence degradation model for long delegation chains to ensure that deeply nested agents have appropriately reduced trust scores.
- MUST NOT: The join operation (union) MUST NOT be used to compute an agent's effective authority from multiple incoming delegations.
