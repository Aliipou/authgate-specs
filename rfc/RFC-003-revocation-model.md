# RFC-003: Revocation Model

**Status:** Draft  
**Category:** Core  
**Depends on:** RFC-001 (Capability Semantics), RFC-002 (Delegation Algebra)  

---

## Abstract

Defines the revocation semantics for the authgate-kernel capability-governance runtime. Revocation is the mechanism by which an authority holder terminates a previously issued delegation, making the revoked capability no longer exercisable by the recipient or any downstream agent. This RFC specifies two primary revocation models (Eager and Lazy), cascading revocation semantics, lease expiration, freeze semantics, and the consistency guarantees and trade-offs associated with each approach.

---

## 1. Background and Motivation

Delegation without revocation is permanent authority transfer. The ability to revoke is a necessary condition for meaningful governance: a principal must be able to terminate an agent's authority when that agent is compromised, misbehaves, or is simply no longer needed. Revocation is therefore as fundamental as delegation in any capability-governance system.

The authgate-kernel faces the additional challenge that agents may be distributed across multiple nodes, with delegation state replicated or sharded. This creates a fundamental tension between:

- **Consistency:** every node immediately learns of a revocation
- **Availability:** agents can continue operating without contacting a central authority
- **Partition tolerance:** the system remains safe during network partitions

This RFC specifies the revocation primitives and their consistency guarantees, allowing implementors to select the appropriate model for their deployment context.

---

## 2. Revocation Models

### 2.1 Eager Revocation

**Definition:** In the eager model, revocation is performed atomically and synchronously. When a principal issues a revocation for delegation edge `e`, the revocation is committed to the delegation store before control is returned to the principal. All subsequent validation checks will immediately see the edge as revoked.

**Semantics:**

```
revoke_eager(edge_id: EdgeId, principal: Principal) -> Result<(), RevocationError>:
    edge = delegation_store.get(edge_id)
    
    if edge is None:
        return Err(EDGE_NOT_FOUND)
    
    if !can_revoke(principal, edge):
        return Err(UNAUTHORIZED_REVOCATION)
    
    // Atomic write — no capability exercise may interleave
    delegation_store.mark_revoked(edge_id, timestamp=now(), revoker=principal)
    
    // Cascade to all downstream edges
    cascade_revoke_eager(edge_id, principal)
    
    return Ok(())
```

**Consistency guarantee:** Strong consistency. After `revoke_eager` returns `Ok`, no agent will successfully exercise any capability derived from the revoked edge, on any node that shares the delegation store.

**Latency:** Higher — requires synchronous writes and cascade propagation before returning.

**Use cases:** Safety-critical revocations (compromised agent, policy violation, emergency shutdown). Preferred for `CRITICAL` and `CATASTROPHIC` capability grants.

### 2.2 Lazy Revocation

**Definition:** In the lazy model, revocation is recorded in the delegation store but propagation to cached authority views is deferred until the next validation event. Agents holding cached grants may continue to exercise capabilities briefly after revocation, bounded by the cache TTL.

**Semantics:**

```
revoke_lazy(edge_id: EdgeId, principal: Principal) -> Result<(), RevocationError>:
    edge = delegation_store.get(edge_id)
    
    if edge is None:
        return Err(EDGE_NOT_FOUND)
    
    if !can_revoke(principal, edge):
        return Err(UNAUTHORIZED_REVOCATION)
    
    // Write revocation record; downstream nodes will observe on next poll
    delegation_store.mark_revoked(edge_id, timestamp=now(), revoker=principal)
    
    // Cascade records written but not synchronously propagated
    cascade_revoke_lazy(edge_id, principal)
    
    return Ok(())
```

**Consistency guarantee:** Eventual consistency. After `revoke_lazy` returns `Ok`, the revocation is durable, but agents on remote nodes may continue to exercise the revoked capability until their cache expires or they re-validate against the delegation store.

**Maximum staleness window:** Bounded by `CACHE_TTL` (recommended maximum: 60 seconds for `HIGH` capabilities; must be 0 for `CRITICAL` and `CATASTROPHIC`).

**Use cases:** Routine delegation expiry, non-urgent revocations, performance-sensitive paths. MUST NOT be used for `CRITICAL` or `CATASTROPHIC` capability grants (those require eager revocation).

---

## 3. Cascading Revocation

### 3.1 Definition

When principal `p` revokes a delegation edge `(u, v, C, B)`, all delegation edges that are transitively downstream of `v` in the delegation graph are also revoked. This is **cascading revocation**.

Formally, let `downstream(v)` denote the set of all agents reachable from `v` in the delegation DAG:

```
downstream(v) = { w | ∃ directed path from v to w in G }
```

When edge `(u, v)` is revoked, for every `w ∈ downstream(v)`, all delegation edges `(v, w, ...)` and all edges issued by `w` to further downstream agents are also revoked.

### 3.2 Cascade Algorithm

```
function cascade_revoke(root_edge_id: EdgeId, mode: RevocationMode, revoker: Principal):
    // BFS/DFS traversal of the delegation subgraph rooted at the revoked edge
    queue = [root_edge_id]
    revoked_set = {}
    
    while queue is not empty:
        edge_id = queue.dequeue()
        
        if edge_id in revoked_set:
            continue  // Already processed — prevent double-visit in diamond DAGs
        
        revoked_set.add(edge_id)
        delegation_store.mark_revoked(edge_id, timestamp=now(), revoker=revoker)
        
        // Find all edges issued by the recipient of this edge
        edge = delegation_store.get(edge_id)
        child_edges = delegation_store.get_outgoing_edges(edge.recipient)
        
        for each child_edge in child_edges:
            if child_edge.id not in revoked_set:
                queue.enqueue(child_edge.id)
    
    // Emit audit event
    audit_log.append(CascadeRevocationEvent {
        root_edge_id: root_edge_id,
        revoked_edges: revoked_set,
        revoker: revoker,
        timestamp: now(),
        mode: mode,
    })
```

### 3.3 Cascade Scope

Cascading revocation is scoped to the **authority subtree** rooted at the revoked edge. An agent with multiple incoming delegation edges from independent principals is NOT fully revoked when only one incoming edge is revoked. Its effective capability set is reduced to the intersection of remaining valid grants (per RFC-002 §5).

**Example:**

```
Principal A ──(WRITE)──> Agent X ──(WRITE)──> Agent Y
Principal B ──(WRITE)──>    ↑
                            └──(READ)──> Agent Z
```

If Principal A revokes their edge to Agent X, Agent X's `WRITE` capability is revoked (and cascades to Agent Y). However, if Principal B's grant to Agent X remains valid and includes `WRITE`, Agent X retains `WRITE` through that path (subject to budget intersection per RFC-002 §5.1).

---

## 4. Revocation Authorization

A revocation is only valid if issued by an authorized revoker. The following principals are authorized to revoke a delegation edge `(u, v, C, B)`:

1. **The issuing principal:** The agent `u` that issued the delegation edge, provided `B.revocable = true`.
2. **Any ancestor principal:** Any agent or human principal with a delegation path to `u` in the graph.
3. **A human principal with `POLICY_MODIFY`:** A registered human principal exercising governance authority.

Machine agents MUST NOT revoke a delegation edge if `B.revocable = false`, unless they hold `POLICY_MODIFY` (which itself requires human grant per RFC-001 §5).

---

## 5. Lease Expiration as Automatic Revocation

A delegation issued with `expires_at: Some(t)` is automatically revoked at time `t`. Lease expiration is semantically equivalent to eager revocation, with the following properties:

- **No explicit revocation action required:** The runtime enforces expiry automatically during capability validation (see RFC-002 §4).
- **Cascade behavior:** Expired leases trigger the same cascading revocation as explicit revocations. All downstream delegations whose authority flows through an expired lease are also invalidated.
- **Clock skew tolerance:** Implementations MUST apply a clock skew tolerance of `CLOCK_SKEW_TOLERANCE` (recommended: 30 seconds) when evaluating `expires_at`. A capability is considered expired if `now() > expires_at + CLOCK_SKEW_TOLERANCE`.
- **Non-renewable by default:** An expired lease CANNOT be renewed by the recipient. The issuing principal must issue a new delegation edge.

### 5.1 Lease Nesting

For a delegation chain where `u` leases to `v` with `expires_at = T_uv`, and `v` leases to `w` with `expires_at = T_vw`, the following invariant MUST hold:

```
T_vw ≤ T_uv
```

A child delegation lease MUST NOT outlast the parent lease. The runtime MUST reject delegations that would violate this invariant with `LEASE_EXCEEDS_PARENT`. See RFC-002 §2.1 for budget attenuation rules.

---

## 6. Freeze Semantics

A **freeze** is an immutable snapshot of a delegation state at a specific point in time. Freeze is distinct from revocation: a frozen delegation cannot be further modified (no new child delegations, no budget modifications), but the existing capability grants within the freeze remain valid.

### 6.1 Freeze Operations

```
freeze(edge_id: EdgeId, freezer: Principal) -> Result<FreezeHandle, FreezeError>:
    edge = delegation_store.get(edge_id)
    
    if !can_freeze(freezer, edge):
        return Err(UNAUTHORIZED_FREEZE)
    
    snapshot = DelegationSnapshot {
        edge_id: edge_id,
        captured_at: now(),
        state: edge.clone(),
        downstream_state: capture_subtree(edge_id),
    }
    
    delegation_store.mark_frozen(edge_id)
    freeze_store.persist(snapshot)
    
    return Ok(FreezeHandle { snapshot_id: snapshot.id })
```

### 6.2 Freeze Properties

- **Immutability:** A frozen delegation edge CANNOT be extended, modified, or used as a source for new child delegations.
- **Validity preservation:** Existing capability grants under a frozen edge remain valid and exercisable, subject to budget limits and lease expiry.
- **Audit utility:** Freeze snapshots provide a forensic record of delegation state at a specific time, useful for incident investigation.
- **Freeze is not revocation:** A frozen edge is NOT revoked. Agents holding frozen delegations may continue to exercise their granted capabilities.
- **Transitivity:** When a delegation edge is frozen, all downstream edges issued by the recipient are also implicitly frozen (no new further delegations may be issued, but existing ones remain valid).

---

## 7. Revocation Propagation in Distributed Settings

In distributed deployments where the delegation store is replicated across multiple nodes, revocation propagation must contend with network partitions and replication lag.

### 7.1 Safe-Failure Mode

The authgate-kernel adopts a **fail-safe default:** in the absence of confirmation that a revocation has propagated, the system defaults to denying capability exercises. Specifically:

```
if (delegation_store_unreachable() || last_sync_age() > MAX_SYNC_AGE):
    // Deny capability exercise — do not permit stale-cache access for HIGH+ capabilities
    return Err(AUTHORITY_STORE_UNREACHABLE)
```

`MAX_SYNC_AGE` MUST be configurable per capability risk level:

| Risk Level | MAX_SYNC_AGE |
|------------|-------------|
| Low | 300 seconds |
| Medium | 120 seconds |
| High | 30 seconds |
| Critical | 0 seconds (no stale cache permitted) |
| Catastrophic | 0 seconds (no stale cache permitted) |

### 7.2 Revocation Propagation Protocol

Revocation events MUST be disseminated using a **push-pull hybrid** protocol:

1. **Push:** Upon revocation, the originating node broadcasts a signed `RevocationNotice` to all known replicas.
2. **Pull:** Each node periodically polls the authoritative delegation store for revocations newer than its last-known timestamp.
3. **Signed notices:** `RevocationNotice` messages MUST be signed by the revoking principal's key. Unsigned or unverifiable revocation notices MUST be discarded.

```
RevocationNotice = {
    edge_id:      EdgeId,
    revoked_at:   Timestamp,
    revoker:      Principal,
    signature:    Signature,   // Signs (edge_id || revoked_at || revoker)
    cascade_ids:  Vec<EdgeId>, // All transitively revoked edges
}
```

### 7.3 Partition Behavior

During a network partition, nodes that cannot reach the authoritative delegation store MUST:

1. Continue to serve `LOW` and `MEDIUM` capability requests if within `MAX_SYNC_AGE`.
2. Deny all `HIGH`, `CRITICAL`, and `CATASTROPHIC` capability requests.
3. Queue all revocation events received locally for replay once the partition heals.
4. On partition recovery, apply all queued revocations before resuming normal operation.

---

## 8. Consistency Guarantees and Trade-offs

| Property | Eager Revocation | Lazy Revocation |
|----------|-----------------|-----------------|
| Revocation latency | Immediate | Eventual (bounded by cache TTL) |
| Post-revocation window | Zero | Up to CACHE_TTL |
| Distributed propagation | Synchronous | Asynchronous |
| Throughput impact | Higher (synchronous cascade) | Lower |
| Partition behavior | Fails safe (blocks on partition) | Continues with stale data |
| Audit completeness | Immediate | Delayed |
| Recommended for | CRITICAL / CATASTROPHIC | LOW / MEDIUM |

### 8.1 Known Limitations

- **TOCTOU window in lazy mode:** Between a lazy revocation write and its propagation, a window exists during which a revoked agent may successfully exercise capabilities. This window is bounded but non-zero.
- **Clock skew:** Lease expiration relies on synchronized clocks. Implementations must configure NTP or equivalent and apply the `CLOCK_SKEW_TOLERANCE` parameter.
- **Cascade cost:** Cascading revocation is O(|downstream(v)|) in time and write amplification. Deep, wide delegation trees may cause significant cascade latency. The `MAX_DELEGATION_DEPTH` bound (RFC-002) limits worst-case cascade to 2^16 nodes, which SHOULD be acceptable given that typical deployments will have much shallower trees.

---

## 9. Normative Requirements

- MUST: Every delegation edge issued with `B.revocable = true` MUST be revocable by the issuing agent or any ancestor principal.
- MUST: Revocation MUST cascade to all downstream delegation edges within the revoked subtree.
- MUST: Cascade revocation MUST be idempotent. Revoking an already-revoked edge MUST NOT error and MUST NOT cause duplicate audit entries.
- MUST: `CRITICAL` and `CATASTROPHIC` capability grants MUST use eager revocation. Lazy revocation is PROHIBITED for these risk levels.
- MUST: Lease expiration MUST be treated as automatic eager revocation with cascading semantics.
- MUST: In distributed deployments, nodes that cannot reach the delegation store within `MAX_SYNC_AGE` MUST deny `HIGH` and above capability exercises.
- MUST NOT: Revocations MUST NOT be undone. Authority may only be re-granted by issuing a new delegation edge.
- SHOULD: All revocation events SHOULD be written to an append-only audit log before the revocation is committed.
