# RFC-001: Capability Semantics

**Status:** Draft  
**Category:** Core  
**Depends on:** None  

---

## Abstract

Defines the closed, finite vocabulary of capability kinds recognized by the freedom-kernel runtime, their semantics, and risk classification.

---

## 1. Capability Taxonomy

A capability is an unforgeable token of authority that grants its holder permission to perform a specific operation on a specific resource. Capabilities are discrete, enumerable, and carry no implicit ordering unless explicitly defined.

### 1.1 Capability Kinds

| Kind | Code | Risk | Description |
|------|------|------|-------------|
| Read | `READ` | Low | Read access to resource content |
| Write | `WRITE` | Medium | Modify resource content |
| Execute | `EXECUTE` | High | Execute resource as code or process |
| Delegate | `DELEGATE` | High | Transfer a subset of held capabilities to another agent |
| Spawn Agent | `SPAWN_AGENT` | Critical | Create a new autonomous agent |
| Network Access | `NETWORK_ACCESS` | Critical | Establish external network connections |
| Model Invoke | `MODEL_INVOKE` | Critical | Invoke an LLM or other ML model |
| Train | `TRAIN` | Critical | Train a model on data |
| Fine-Tune | `FINE_TUNE` | Critical | Modify model weights |
| Memory Access | `MEMORY_ACCESS` | Medium | Access agent working memory |
| Tool Invoke | `TOOL_INVOKE` | High | Invoke an external tool (default High; tool-specific risk applies) |
| System Prompt Edit | `SYSTEM_PROMPT_EDIT` | Critical | Modify the system prompt of an agent |
| Policy Modify | `POLICY_MODIFY` | Catastrophic | Modify the governance policy itself |
| IPC Send | `IPC_SEND` | High | Send inter-process communication message |
| IPC Receive | `IPC_RECEIVE` | High | Receive inter-process communication message |
| Consume Quota | `CONSUME_QUOTA` | Medium | Consume a resource quota (tokens, API calls, etc.) |
| Enter Domain | `ENTER_DOMAIN` | High | Cross a trust domain boundary |

### 1.2 Risk Classification

| Risk | Meaning | Default policy |
|------|---------|----------------|
| Low | Bounded impact; reversible | Permit with logging |
| Medium | Moderate impact; may affect shared state | Permit with audit |
| High | Significant impact; potentially irreversible | Require explicit delegation |
| Critical | Severe impact; system-wide consequences | Require human-in-loop delegation |
| Catastrophic | Irreversible; affects the governance layer itself | Prohibited without explicit principal grant |

---

## 2. Capability Operations

Capabilities are operated upon through the following transfer operations:

| Operation | Semantics |
|-----------|-----------|
| `DELEGATE` | Temporary subset — delegator retains authority |
| `TRANSFER` | Ownership move — delegator loses authority |
| `ATTENUATE` | Strictly weaker capability — fewer permissions than source |
| `CLONE` | Duplicate authority — both hold equivalent claims |
| `LEASE` | Time-bound delegation — expires automatically |
| `REVOKE` | Invalidate — removes capability from holder |

---

## 3. Attenuation Invariant

For any delegation chain `p₀ → p₁ → ... → pₙ`:

```
∀i: cap(pᵢ) ⊆ cap(pᵢ₋₁)
```

This must be enforced atomically at delegation time. A delegation that would violate this invariant MUST be rejected with `ATTENUATION_VIOLATION`.

---

## 4. Capability Composition

Capabilities do not implicitly compose. An agent holding `READ` and `WRITE` on separate resources does not implicitly hold any combined capability. Capabilities are per-resource, per-operation, and non-transitive by default.

Exception: `DELEGATE` over a resource implicitly requires the agent to hold the delegated operations. An agent cannot delegate `WRITE` on a resource without holding `WRITE` on that resource.

---

## 5. Normative Requirements

- MUST: All capability kinds in use MUST be enumerated at compile time. No open extension point is permitted in the TCB.
- MUST: `POLICY_MODIFY` and `SYSTEM_PROMPT_EDIT` MUST require explicit human principal grant. They MUST NOT be delegatable by machines.
- SHOULD: Implementations SHOULD log all `CRITICAL` and `CATASTROPHIC` capability invocations to an append-only audit trail.
- MUST NOT: A capability MAY NOT be self-granted. Authority must flow from a registered human principal.
