# RFC-004: Distributed Authority Resolution

**Status:** Draft  
**Category:** Distribution  
**Depends on:** RFC-001 (Capability Semantics), RFC-002 (Delegation Algebra), RFC-003 (Revocation Model)  

---

## Abstract

Defines how capability authority is resolved across distributed nodes in the authgate-kernel runtime. This RFC specifies the format and validation of signed capability tokens, procedures for handling stale permissions under clock skew and network partitions, Byzantine-resilient authority validation, and the structure of the immutable audit chain that records delegation, revocation, and action attestation history.

---

## 1. Motivation

In a single-node deployment, authority resolution is straightforward: a central delegation store is queried synchronously. In distributed deployments, agents may execute on nodes separated by network boundaries, replication lag, and potential Byzantine faults. This RFC defines the mechanisms that allow authority to be validated efficiently without requiring round-trips to a central store on every capability exercise, while preserving the invariants defined in RFC-002 and RFC-003.

The design draws on principles from object-capability systems (E language, Caja), token-based delegation (Macaroons, SPIFFE/SPIRE), and Byzantine fault-tolerant consensus (PBFT, HotStuff) — but is tailored specifically to the capability-governance semantics of the authgate-kernel.

---

## 2. Signed Capability Tokens

### 2.1 Token Structure

A **Signed Capability Token (SCT)** is a tamper-evident credential that encodes a delegation claim. SCTs allow agents to present their authority to remote nodes without requiring the remote node to contact the delegation store at exercise time, subject to the staleness constraints in RFC-003 §7.1.

```
SignedCapabilityToken = {
    header: TokenHeader,
    claims: TokenClaims,
    signature: Signature,       // Signs canonical encoding of (header || claims)
    delegation_proof: DelegationProof,
}

TokenHeader = {
    version:       u8,           // Token format version (currently 1)
    token_id:      UUID,         // Globally unique token identifier
    issued_at:     Timestamp,
    algorithm:     SignatureAlg, // e.g., Ed25519, ECDSA-P256
    issuer_key_id: KeyId,        // References issuer's public key in the key registry
}

TokenClaims = {
    subject:           AgentId,        // The agent to whom authority is granted
    issuer:            AgentId,        // The agent issuing this token
    capabilities:      Vec<CapGrant>,  // Capability grants encoded in this token
    budget:            AuthorityBudget,
    resource_scope:    ResourceScope,  // Resources to which capabilities apply
    delegation_chain:  Vec<TokenId>,   // Ordered chain of parent token IDs
}

CapGrant = {
    kind:      CapabilityKind,    // From RFC-001 §1.1
    operations: Vec<Operation>,   // Specific operations permitted
}
```

### 2.2 Token Chaining

SCTs are chained: each token references the `token_id` of its parent in `delegation_chain`. This allows a verifier to reconstruct the full delegation path without querying the central store. The chain MUST be ordered from trust root to immediate parent.

**Attenuation enforcement:** A token MUST NOT grant capabilities that exceed those in any of its parent tokens. A verifier MUST walk the chain and verify the attenuation invariant at each step (RFC-002 Rule D1).

### 2.3 Token Validation Algorithm

```
function validate_token(token: SignedCapabilityToken, cap: Capability, resource: Resource)
    -> Result<(), ValidationError>:

    // Step 1: Verify signature
    if !verify_signature(token.header.issuer_key_id, token.signature, token):
        return Err(INVALID_SIGNATURE)
    
    // Step 2: Check expiry
    if token.claims.budget.expires_at is Some(t) and now() > t + CLOCK_SKEW_TOLERANCE:
        return Err(TOKEN_EXPIRED)
    
    // Step 3: Check revocation
    if revocation_store.is_revoked(token.header.token_id):
        return Err(TOKEN_REVOKED)
    
    // Step 4: Verify capability is in grant
    if cap ∉ token.claims.capabilities:
        return Err(CAPABILITY_NOT_GRANTED)
    
    // Step 5: Verify resource scope
    if resource ∉ token.claims.resource_scope:
        return Err(RESOURCE_OUT_OF_SCOPE)
    
    // Step 6: Validate parent chain (recursively)
    for each parent_token_id in token.claims.delegation_chain:
        parent_token = token_store.get(parent_token_id)
        if parent_token is None:
            return Err(PARENT_TOKEN_NOT_FOUND(parent_token_id))
        
        // Attenuation check: this token must not exceed parent
        if !is_attenuated(token.claims, parent_token.claims):
            return Err(ATTENUATION_VIOLATION)
    
    // Step 7: Check depth
    if |token.claims.delegation_chain| > MAX_DELEGATION_DEPTH:
        return Err(DEPTH_EXCEEDED)
    
    return Ok(())
```

### 2.4 Token Issuance

Tokens are issued by the authgate-kernel authority service upon successful delegation (RFC-002). Token issuance requires:

1. The issuing agent holds a valid SCT granting `DELEGATE` for the capability being delegated.
2. The new token's claims satisfy all attenuation rules (RFC-002 D1).
3. The delegation DAG constraints are satisfied (RFC-002 §1.2).
4. The issuer's private key is accessible to the authority service (HSM or equivalent).

Tokens MUST be signed using a key registered in the **Key Registry** (see §2.5). Self-signed tokens from unregistered keys MUST be rejected.

### 2.5 Key Registry

The Key Registry is an append-only log of principal and agent public keys, with associated metadata:

```
KeyRegistryEntry = {
    key_id:       KeyId,
    public_key:   PublicKey,
    owner:        AgentId,
    registered_at: Timestamp,
    revoked_at:   Option<Timestamp>,
    key_type:     KeyType,   // PRINCIPAL | AGENT | DELEGATION_SERVICE
}
```

Key registration requires a human principal. Machine agents cannot register new keys without an existing valid delegation. Key revocation cascades to all tokens signed by the revoked key.

---

## 3. Stale Permission Handling

### 3.1 Clock Skew

Distributed systems cannot assume perfectly synchronized clocks. The authgate-kernel defines the following clock skew handling:

**`CLOCK_SKEW_TOLERANCE`**: A global configuration parameter (recommended default: 30 seconds) applied when evaluating `expires_at` timestamps. A token is considered expired if:

```
now() > expires_at + CLOCK_SKEW_TOLERANCE
```

To prevent clock skew from being exploited as a privilege-escalation vector, `CLOCK_SKEW_TOLERANCE` MUST be bounded:

```
0 ≤ CLOCK_SKEW_TOLERANCE ≤ 60 seconds
```

Implementations MUST reject configurations with `CLOCK_SKEW_TOLERANCE > 60 seconds`.

### 3.2 Handling Partition and Replication Lag

When an agent presents an SCT to a remote node, that node must check the revocation store. If the revocation store is unreachable, the node applies the risk-level-based staleness policy from RFC-003 §7.1.

**Proactive token pinning:** For time-critical agents, the runtime MAY proactively fetch and locally cache the revocation status of all tokens in the agent's current grant set, refreshing at an interval no greater than `MAX_SYNC_AGE / 2` for each risk level.

### 3.3 Stale Grant Attack Mitigation

A **stale grant attack** occurs when an agent presents a valid, non-expired SCT for a capability that has been revoked at the delegation store but whose revocation has not yet propagated to the verifying node.

Mitigations:

1. **Short-lived tokens:** For `HIGH` capabilities, `expires_at` SHOULD be set to no more than 300 seconds from issuance, limiting the maximum staleness window.
2. **Online revocation check:** For `CRITICAL` and `CATASTROPHIC` capabilities, verifiers MUST perform an online check against the revocation store before permitting exercise, regardless of token validity.
3. **Nonce binding:** Tokens MAY include a challenge nonce issued by the verifier, preventing replay of captured tokens.

---

## 4. Byzantine-Resilient Authority Validation

### 4.1 Threat Model

The authgate-kernel assumes the following adversary capabilities in a Byzantine setting:

- A Byzantine agent may forge, replay, or modify capability claims.
- A Byzantine node may lie about revocation status.
- A Byzantine clock may report incorrect timestamps.
- Up to `f` Byzantine nodes exist in a cluster of `3f + 1` nodes (standard BFT threshold).

### 4.2 Byzantine Validation Protocol

For `CRITICAL` and `CATASTROPHIC` capability exercises, the verifier MUST obtain confirmation from a quorum of delegation store replicas before permitting the exercise.

```
function validate_critical_capability(token: SCT, cap: Capability) -> Result<(), ValidationError>:
    quorum_size = (2 * f) + 1  // where f = max Byzantine faults tolerated
    
    responses = []
    for each replica in delegation_store_replicas:
        response = replica.check_revocation(token.header.token_id, timeout=QUORUM_TIMEOUT)
        responses.append(response)
    
    valid_responses = [r for r in responses if r.is_valid_and_signed()]
    
    if |valid_responses| < quorum_size:
        return Err(INSUFFICIENT_QUORUM)
    
    // Check for conflicting responses (Byzantine equivocation)
    revoked_count = count(r for r in valid_responses if r.revoked)
    valid_count = count(r for r in valid_responses if !r.revoked)
    
    if revoked_count >= (f + 1):
        return Err(TOKEN_REVOKED)  // At least one honest node says revoked
    
    if valid_count >= quorum_size:
        return Ok(())
    
    return Err(QUORUM_AMBIGUOUS)
```

**Quorum signing:** Each replica response MUST be signed by the replica's key. Unsigned or unverifiable responses MUST be excluded from quorum counting.

### 4.3 Equivocation Detection

A Byzantine replica may return different revocation statuses to different querying agents (equivocation). The runtime handles equivocation as follows:

- If a node receives conflicting signed responses from the same replica for the same token, the replica is marked as potentially Byzantine and flagged for operator review.
- The conflicting responses are logged as `EQUIVOCATION_EVENT` in the audit trail.
- The node conservatively treats the token as revoked when equivocation is detected.

---

## 5. Audit Chain

### 5.1 Purpose

The audit chain is an append-only, cryptographically linked record of all authority-relevant events in the authgate-kernel. It serves as the primary tool for:

- **Post-incident forensics:** Reconstructing what authority was held, exercised, and revoked at any point in time.
- **Compliance attestation:** Demonstrating that the governance rules were followed.
- **Byzantine accountability:** Identifying malicious or faulty nodes through inconsistency analysis.

### 5.2 Audit Event Types

```
AuditEvent = 
    | DelegationEvent   { edge_id, issuer, recipient, capabilities, budget, timestamp }
    | RevocationEvent   { edge_id, revoker, cascade_ids, mode, timestamp }
    | ExerciseEvent     { agent_id, capability, resource, token_id, outcome, timestamp }
    | LeaseExpiryEvent  { edge_id, expires_at, cascade_ids, timestamp }
    | KeyRegistration   { key_id, owner, timestamp }
    | KeyRevocation     { key_id, revoker, timestamp }
    | PolicyModification { change_id, principal, before_hash, after_hash, timestamp }
    | EquivocationEvent { replica_id, token_id, conflicting_responses, timestamp }
```

### 5.3 Audit Chain Structure

The audit chain is a hash-linked log, similar in structure to a blockchain but without consensus requirements (the audit chain itself is authoritative; it does not need Byzantine agreement, as it is write-once and append-only).

```
AuditBlock = {
    block_id:      u64,          // Monotonically increasing
    events:        Vec<AuditEvent>,
    previous_hash: Hash,         // SHA-256 of previous block's canonical encoding
    merkle_root:   Hash,         // Merkle root of events in this block
    timestamp:     Timestamp,
    author:        NodeId,
    signature:     Signature,    // Signs (block_id || events || previous_hash || timestamp)
}
```

Blocks MUST be append-only. No modification, deletion, or reordering of blocks is permitted. Any deviation from the expected `previous_hash` chain constitutes evidence of tampering.

### 5.4 Immutable Delegation History

Every delegation issuance MUST generate a `DelegationEvent` in the audit chain before the delegation takes effect. This means a delegation that is not recorded in the audit chain MUST be treated as invalid, even if the agent presents a valid-seeming SCT.

**Audit-first ordering:** The write ordering is:

1. Write `DelegationEvent` to audit chain.
2. Persist the delegation edge to the delegation store.
3. Issue the SCT to the delegatee.

If step 1 fails, steps 2 and 3 MUST NOT proceed.

### 5.5 Revocation History

Every revocation MUST generate a `RevocationEvent` before the revocation takes effect. For cascading revocations, all cascade target edge IDs MUST be listed in the `cascade_ids` field of a single `RevocationEvent`. This provides a complete, single-event record of the full cascade.

### 5.6 Action Attestations

For `CRITICAL` and `CATASTROPHIC` capability exercises, the runtime generates an `ExerciseEvent` attestation signed by the exercising node:

```
ExerciseAttestation = {
    agent_id:    AgentId,
    capability:  CapabilityKind,
    resource:    ResourceId,
    token_id:    TokenId,
    outcome:     PERMITTED | DENIED,
    reason:      Option<ValidationError>,  // Present if DENIED
    timestamp:   Timestamp,
    node_id:     NodeId,
    signature:   Signature,
}
```

This provides non-repudiation for high-risk actions: a node cannot later deny having permitted or denied a specific capability exercise.

---

## 6. Normative Requirements

- MUST: All SCTs MUST be signed by a key registered in the Key Registry. Tokens with unregistered or revoked signing keys MUST be rejected.
- MUST: Token validation MUST verify the full delegation chain, not only the immediate token.
- MUST: For `CRITICAL` and `CATASTROPHIC` capabilities, validation MUST include an online quorum check against the revocation store.
- MUST: `CLOCK_SKEW_TOLERANCE` MUST NOT exceed 60 seconds.
- MUST: Every delegation, revocation, and `CRITICAL`/`CATASTROPHIC` capability exercise MUST be recorded in the audit chain before taking effect.
- MUST: The audit chain MUST be append-only. Any hash chain break MUST be treated as a tampering event and trigger an alert.
- MUST: Byzantine equivocation MUST be logged and the equivocating token treated as revoked.
- SHOULD: Short-lived tokens (≤ 300 seconds) SHOULD be used for `HIGH` capability grants.
- MUST NOT: Token issuance MUST NOT proceed if the `DelegationEvent` write to the audit chain fails.
