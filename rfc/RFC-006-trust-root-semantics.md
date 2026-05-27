# RFC-006: Trust Root Semantics

**Status:** Draft  
**Category:** Core  
**Depends on:** RFC-001 (Capability Semantics), RFC-002 (Delegation Algebra), RFC-005 (Capability Attenuation)  

---

## Abstract

Defines the semantics of trust roots in the authgate-kernel capability-governance system. Trust roots are the mandatory anchors of all authority: every capability grant must trace back to a registered human principal. This RFC specifies what constitutes a trust root, how trust domains partition the authority namespace, the rules for cross-domain delegation, the hierarchy and inheritance structure of trust domains, and the system's behavior when a trust root is compromised. Principal authentication is explicitly out of scope; this RFC defines a structural invariant, not an authentication mechanism.

---

## 1. Human Principals as Mandatory Trust Root

### 1.1 Axiom A4: Human Ownership

The authgate-kernel is founded on **Axiom A4**:

> Every machine agent in the system has exactly one registered human owner. All authority exercised by a machine agent must be traceable to a delegation chain rooted at a registered human principal.

This axiom is structural, not behavioral: it is a constraint on the topology of the delegation graph (RFC-002 §1.1), not on the runtime behavior of principals. The runtime enforces this invariant by rejecting any capability exercise for which no valid delegation chain reaching a registered human principal can be found.

### 1.2 Principal Registration

A **human principal** is an entity registered in the **Principal Registry** — an authoritative, append-only record of human identities with associated cryptographic keys. A principal registration record has the form:

```
PrincipalRecord = {
    principal_id:   PrincipalId,    // Globally unique identifier
    display_name:   String,         // Non-authoritative human-readable name
    public_keys:    Vec<KeyId>,     // References to Key Registry entries
    registered_at:  Timestamp,
    registered_by:  Option<PrincipalId>,  // None for bootstrap principals
    domain:         TrustDomainId,  // The domain this principal administers
    status:         ACTIVE | SUSPENDED | DEREGISTERED,
}
```

**Key invariant:** A principal's `public_keys` field MUST reference entries in the Key Registry (RFC-004 §2.5). A principal with no valid keys is treated as deregistered for the purpose of authority validation.

### 1.3 Bootstrap Principals

The system requires at least one principal to exist before any others can be registered. **Bootstrap principals** are principals registered through an out-of-band initialization procedure (e.g., hardware-backed key ceremony, HSM provisioning). Bootstrap principals have `registered_by = None`. All other principals MUST have `registered_by` referencing an active principal in the same or an ancestor trust domain.

There MUST be at least one bootstrap principal per trust domain.

### 1.4 Authority as a Derived Property

Machine agents do not hold native authority. Authority is always derived through delegation from a human principal. Formally:

```
∀ agent a ∈ A \ P: holds(a) ≠ ∅  ⟹  ∃ p ∈ P, ∃ chain from p to a in G
```

Any capability exercise by a machine agent for which no such chain exists MUST be denied with `NO_PRINCIPAL_ANCHOR`.

---

## 2. Trust Domains

### 2.1 Definition

A **trust domain** `D = (id, policy, principals, agents)` is an isolated authority namespace within the authgate-kernel system:

```
TrustDomain = {
    domain_id:       TrustDomainId,    // Unique domain identifier
    name:            String,
    policy:          DomainPolicy,     // Governance rules for this domain
    parent:          Option<TrustDomainId>,  // Enclosing domain (None for root)
    principals:      Set<PrincipalId>, // Human principals who administer this domain
    agents:          Set<AgentId>,     // Machine agents resident in this domain
    cross_domain_grants: Vec<CrossDomainGrant>,  // Explicit outbound grants
    created_at:      Timestamp,
    status:          ACTIVE | FROZEN | DISSOLVED,
}
```

### 2.2 Domain Isolation

By default, trust domains are **isolated**: a capability grant made within domain `D₁` does not extend to agents in domain `D₂`, even if those agents are otherwise connected to the same infrastructure. Isolation is enforced at the delegation graph level: delegation edges MUST be tagged with source and destination domains, and cross-domain edges require an explicit `CrossDomainGrant`.

**Rationale:** Domain isolation limits the blast radius of a compromised domain. An attacker who fully compromises `D₁` (including all human principals) cannot automatically extend their authority into `D₂`.

### 2.3 Domain Policy

Each domain has a `DomainPolicy` that governs the default rules for capability grants within the domain:

```
DomainPolicy = {
    default_max_delegation_depth:  u8,    // ≤ MAX_DELEGATION_DEPTH (16)
    default_lease_duration:        Duration,
    prohibited_capabilities:       Set<CapabilityKind>,  // Always denied in this domain
    require_audit_for:             Set<RiskLevel>,
    require_human_approval_for:    Set<CapabilityKind>,
    cross_domain_delegation_allowed: bool,
}
```

Domain policy is set by the domain's principals and MUST satisfy any constraints imposed by the parent domain's policy.

---

## 3. Trust Domain Hierarchy

### 3.1 Hierarchical Structure

Trust domains are organized in a **rooted tree** hierarchy. The root domain (also called the **system domain**) is established during bootstrap and has no parent. All other domains have exactly one parent domain.

```
system_domain (root)
├── domain_A
│   ├── domain_A1
│   └── domain_A2
└── domain_B
    └── domain_B1
```

The hierarchy defines the scope of domain policy authority: a parent domain's policy MUST be at least as restrictive as any child domain's policy. A child domain CANNOT be granted capabilities that the parent domain does not itself permit.

**Formally:** For parent domain `Dp` and child domain `Dc ∈ Dp.children`:

```
Dc.policy.prohibited_capabilities ⊇ Dp.policy.prohibited_capabilities
Dc.policy.default_max_delegation_depth ≤ Dp.policy.default_max_delegation_depth
```

### 3.2 Policy Inheritance

A child domain inherits its parent domain's policy as a floor: the child's policy MUST be at least as restrictive as the parent's. The child MAY impose additional restrictions but CANNOT relax the parent's constraints.

**Inheritance rule:** When the authgate-kernel evaluates a capability exercise by an agent in domain `Dc`, it applies the union of all policies along the path from `Dc` to the root domain. Specifically, the effective prohibited capability set is:

```
effective_prohibited(Dc) = ⋃ { D.policy.prohibited_capabilities | D is an ancestor of Dc or D = Dc }
```

A capability that is prohibited at any level of the domain hierarchy MUST be denied for all agents in the subtree below that level.

### 3.3 Domain Creation

A new trust domain MUST be created by a human principal with authority in the parent domain. Domain creation requires:

1. A registered human principal in the parent domain issuing a `CreateDomain` governance action.
2. The new domain's policy satisfying the inheritance constraints.
3. At least one bootstrap principal registered in the new domain.

Domain creation MUST be recorded in the audit chain (RFC-004 §5.2) as a `PolicyModification` event.

---

## 4. Cross-Domain Delegation Rules

### 4.1 Cross-Domain Grant

A **cross-domain grant** is an explicit authorization for authority to flow from domain `D₁` to domain `D₂`. It must be issued by a human principal in `D₁` (or a common ancestor domain):

```
CrossDomainGrant = {
    grant_id:       GrantId,
    source_domain:  TrustDomainId,
    target_domain:  TrustDomainId,
    capabilities:   Vec<CapGrant>,    // Subset allowed to cross
    conditions:     Vec<Condition>,   // Optional predicates (e.g., time bounds)
    issued_by:      PrincipalId,
    expires_at:     Option<Timestamp>,
    revocable:      bool,
}
```

### 4.2 Cross-Domain Delegation Invariants

**Invariant CD1 — Explicit Authorization Required:**

An agent in `D₁` MUST NOT delegate capabilities to an agent in `D₂` unless a valid `CrossDomainGrant` from `D₁` to `D₂` (or a common ancestor authorizing the crossing) exists. Attempts to delegate across domains without authorization MUST be rejected with `CROSS_DOMAIN_UNAUTHORIZED`.

**Invariant CD2 — Capability Restriction:**

The capabilities crossing a domain boundary MUST be a subset of those specified in the `CrossDomainGrant`:

```
C_cross ⊆ CrossDomainGrant.capabilities
```

**Invariant CD3 — Principal Anchoring Preserved:**

After a cross-domain delegation, the receiving agent in `D₂` MUST still satisfy the principal anchoring requirement (§1.4). The delegation chain MUST trace to a principal in `D₁` (via the cross-domain grant) or to a principal in a common ancestor domain. The cross-domain grant itself anchors to the human principal who issued it.

**Invariant CD4 — No Upward Escalation:**

Cross-domain delegation MUST NOT be used to escalate capabilities upward in the domain hierarchy. Specifically, an agent in a child domain MUST NOT use cross-domain delegation to obtain capabilities from a parent domain that exceed what was explicitly delegated down to the child domain.

**Invariant CD5 — Domain Policy Compliance:**

The delegated capabilities MUST NOT include any capability that is prohibited by either the source domain's policy or the target domain's policy.

```
C_cross ∩ effective_prohibited(D₁) = ∅
C_cross ∩ effective_prohibited(D₂) = ∅
```

### 4.3 Sibling Domain Delegation

Two domains `D₁` and `D₂` under a common parent `Dp` may delegate to each other only if:

1. A `CrossDomainGrant` exists (issued by a principal in `Dp` or a common ancestor), AND
2. Both `D₁` and `D₂`'s policies permit cross-domain delegation (`cross_domain_delegation_allowed = true`), AND
3. The parent domain `Dp`'s policy does not prohibit the capabilities being crossed.

---

## 5. Trust Root Compromise: Explicit Limitation

### 5.1 Scope of the Invariant

The trust root semantics defined in this RFC are **structural invariants** — they define what the system enforces, not how the system detects or prevents principal compromise. The authgate-kernel does NOT provide:

- Principal authentication (verifying that the human behind a key is who they claim to be)
- Multi-factor authentication for principal actions
- Insider threat detection
- Social engineering resistance

These are concerns for the authentication and identity layer, which is explicitly out of scope for this RFC.

### 5.2 When a Trust Root is Compromised

If a human principal's private key is compromised, the attacker gains the principal's full authority, including the ability to issue new delegations, revoke existing ones, and modify domain policy. The authgate-kernel cannot distinguish between actions by the legitimate principal and actions by the attacker once the key is compromised.

**What the system CAN provide:**

1. **Audit trail:** All actions taken with the compromised key are recorded in the append-only audit chain (RFC-004 §5). Post-compromise forensics can identify all delegations issued and capabilities exercised.

2. **Key revocation:** Upon discovering the compromise, another authorized principal (in the same or ancestor domain) can revoke the compromised principal's keys in the Key Registry. This cascades to all delegations issued by the compromised principal (RFC-003 §3).

3. **Domain isolation:** Compromise of a principal in `D₁` does not automatically extend to `D₂` (§2.2). Authority is constrained by domain boundaries.

4. **Bounded blast radius:** Because delegation depth is bounded (`MAX_DELEGATION_DEPTH = 16`) and domains are isolated, the scope of damage from a single compromised principal is structurally limited.

**What the system CANNOT provide:**

1. **Retroactive protection:** Actions taken by the attacker before key revocation are recorded as legitimate exercises of authority by the principal.

2. **Prevention of malicious revocation:** A compromised principal can revoke the authority of legitimate downstream agents, disrupting operations.

3. **Prevention of malicious delegation:** A compromised principal can issue delegations to attacker-controlled agents before revocation.

### 5.3 Recommended Mitigations (Informative)

The following mitigations are recommended but not enforced by the authgate-kernel itself:

- **M-of-N principal authorization:** For `CRITICAL` and `CATASTROPHIC` capability grants, require approval from `M` out of `N` human principals (multi-party authorization). This limits the blast radius of a single compromised principal.
- **Hardware security modules:** Principal signing keys SHOULD be held in HSMs or equivalent hardware-backed key stores. Software-only key storage is discouraged.
- **Key rotation:** Principal keys SHOULD be rotated periodically, with the rotation recorded in the Key Registry and the audit chain.
- **Canary principals:** Dormant principals with known-sensitive capabilities can serve as canaries; unexpected use of these capabilities triggers an alert.

---

## 6. Normative Requirements

- MUST: Every machine agent MUST have a traceable delegation path to at least one registered human principal. Capability exercises by agents without this path MUST be denied.
- MUST: Trust domains MUST be isolated by default. Cross-domain delegation requires an explicit `CrossDomainGrant` issued by a human principal.
- MUST: Child domain policies MUST be at least as restrictive as their parent domain's policy in all dimensions.
- MUST: Domain creation MUST require a human principal in the parent domain and MUST be recorded in the audit chain.
- MUST: Cross-domain delegated capabilities MUST be a subset of those authorized by the `CrossDomainGrant`.
- MUST: Cross-domain grants MUST NOT permit capabilities prohibited by either the source or target domain's effective policy.
- MUST: Upon key revocation, all delegations issued by the revoked key MUST cascade-revoke per RFC-003 §3.
- MUST NOT: The authgate-kernel MUST NOT assume that a principal's identity has been authenticated. Authentication is out of scope. The structural invariants hold regardless of authentication layer behavior.
- SHOULD: Deployments with `CRITICAL` or `CATASTROPHIC` capability grants SHOULD implement M-of-N principal authorization as a mitigation against trust root compromise.
- SHOULD: Principal signing keys SHOULD be held in hardware-backed key stores.
