# Legitimacy ⊥ Authority: A Two-Question Architecture for Governing Autonomous-Agent Tool Execution

**Draft — technical report. Author: Ali Pourrahim. Theory of Freedom (normative layer): Mohammad Ali Jannat Khah Doust.**

---

## Abstract

Autonomous software agents increasingly act on the world through tools — shells,
HTTP clients, cloud APIs, robots. The prevailing safety approach trains the model
to refuse bad actions (RLHF, Constitutional AI) or wraps calls in a denylist of
patterns. Both are *learned or enumerated* controls: they fail on the case nobody
trained or listed. We describe an alternative that is *deterministic and
fail-closed*, built from a single architectural separation we call **Legitimacy ⊥
Authority**. Every proposed tool call passes two independent questions, in order:
a **legitimacy** question — *should this happen at all?* — which may only **deny**;
and an **authority** question — *does this actor hold the capability?* — which
grants only within legitimacy. A single kernel signs each authorized decision,
binds it to the exact action content, mints a one-time capability token, and
appends the decision to a tamper-evident log before any effect runs. We give the
invariants, the threat model, the reference implementation, and — importantly —
an honest account of what is and is not demonstrated. We do **not** claim this
beats existing policy engines or alignment methods; that comparison is unmeasured
and stated as an open problem.

---

## 1. Introduction

The AI-agent safety conversation is dominated by one framing: make the model
*want* to do the right thing. That framing produces controls that are as reliable
as the training distribution and no more. A model fine-tuned to refuse "delete the
production database" may still call `rm -rf` when the request is obfuscated,
role-played, or simply novel. Pattern-based guardrails have the dual failure:
they miss the attack that isn't in the list, and they over-block the benign
string that happens to match.

This report takes a different position: **the question of whether a machine may
act is not a question about the machine's intelligence; it is a question about
authority and ownership.** A tool call is legitimate or not for reasons that are
independent of how the request was phrased, and an actor either holds a capability
or does not. Both are decidable by a small, auditable mechanism that does not
learn and does not guess.

The contribution is not a new cryptographic primitive or a new policy language.
It is an **architectural separation** — Legitimacy ⊥ Authority — and a claim that
this separation, enforced by structure rather than convention, gives a system that
fails safe by construction. We report the design, a reference implementation
across a set of small components, the security invariants those components
preserve under composition, and the boundaries of the evidence.

---

## 2. Background and problem statement

An agent runtime receives a proposed action `a` (a tool name plus arguments) from
a policy/model and must decide whether to execute it. Three properties are
desirable and usually in tension:

1. **Soundness** — no action executes that violates the governing rules.
2. **Auditability** — every decision is recorded, attributable, tamper-evident.
3. **Neutrality** — the enforcement mechanism does not itself encode a worldview;
   the normative rules are supplied as policy and can be swapped.

Learned safety (RLHF, CAI) achieves none of these as *guarantees* — it shifts a
distribution. Policy engines (OPA/Rego, AWS Cedar) achieve (1)–(3) for
*authorization* but do not, by themselves, separate the prior question of whether
the action is legitimate at all from whether the actor is authorized to perform
it. That conflation is the gap this work targets.

---

## 3. Architecture: Legitimacy ⊥ Authority

### 3.1 The two questions

```
Request
  → LEGITIMACY   "should this happen at all?"      may only DENY
  → AUTHORITY    "does this actor hold the cap?"    grants only within legitimacy
  → KERNEL       verdict + signature + one-time capability token
  → PEP          execute the effect ONLY against a signed, bound decision + unspent token
  → AUDIT        append one hash-chained, tamper-evident record (before the effect)
```

**Legitimacy** is a policy function `L(a) → (ok, reason)`. It answers whether the
action is permissible in principle — independent of who is asking. Crucially it
returns *only* a boolean and a reason: a `False` refuses the action before the
authority kernel is ever consulted; a `True` grants *nothing*, it merely permits
the question to proceed.

**Authority** is the kernel's job: given identity and capability, decide
`ALLOW / DENY / TRANSFORM`, sign the decision, bind it to the action's content
fingerprint, and mint a one-time token.

### 3.2 The invariant

> **Legitimacy may never grant authority. Authority may never override a
> legitimacy denial.**

This is enforced *structurally*, not by discipline. Because `L` returns only
`(ok, reason)` and runs strictly before the kernel, a denied action cannot be
resurrected downstream, and a permitted action still has to win the authority
decision on its own merits. The two layers compose through a data contract (a JSON
boundary), not shared code, so neither can reach into the other's decision.

### 3.3 Mechanism vs. policy

The kernel is **value-neutral**. It enforces whatever legitimacy policy is
injected — a regulation (GDPR, the EU AI Act), an industry framework (NIST AI RMF,
ISO 42001), or a research theory. The normative content lives *on top* as policy;
the mechanism below it decides nothing on its own. This is the classic
mechanism/policy split applied to agent governance, and it is what lets the same
kernel serve incompatible worldviews without modification.

### 3.4 A normative instance: the Theory of Freedom

One concrete normative instance is the *Theory of Freedom* (Jannat Khah Doust): a
rights-ontology in which ownership flows God → human → machine, machines hold only
*delegated* property rights within their human owner's scope, and an action is
`forbidden` if it violates property rights, coerces, deceives, asserts machine
sovereignty over a human, or bypasses the verifier. Two points matter for this
report:

- **The theory spans both layers, not just legitimacy.** Its executable form is
  *not* only the legitimacy filter (ownership/consent/verifier). The theory's own
  execution structure also specifies the *authority* side — delegated machine
  property rights, a tool-permission layer, and runtime enforcement — which is the
  AuthGate role. Both the legitimacy gate and the authority gate are the theory
  made executable; the FDK↔AuthGate pair, in that order, *is* the theory's pipeline
  (`Ownership → Consent → Verifier → … → Runtime Guard → Execution`).
- **Independence is an open, reopened question — not settled.** An earlier analysis
  did not demonstrate the theory's independence from Nozick (side-constraints),
  Pettit (non-domination), and Sen (capability/exit); that result was explicitly
  *undetermined, not refuted, and reopenable*. New evidence has since been
  introduced (a green-team independence defense and a killer/predictive-test
  program), so the status is **under active re-evaluation**. This report neither
  asserts a closed negative result nor claims independence is proven; it treats the
  theory as an injected policy whose normative standing is out of scope here.

---

## 4. Threat model

We assume an adversary who controls the *content* of proposed actions (a
compromised or jailbroken policy/model) but not the kernel process, its signing
key, or the audit log's storage integrity. The adversary may:

- propose arbitrary, obfuscated, or novel malicious actions;
- attempt to replay a previously authorized decision against a different action;
- attempt to mint or forge a capability token;
- attempt to make an advisory/enrichment component grant authority;
- attempt to cause an effect without a corresponding audit record.

Out of scope: kernel-key compromise, host compromise, side channels, and
availability/DoS. "Correctness = containment, not truth": the goal is to fail safe
under a present attacker, not to determine ground-truth intent.

---

## 5. Security properties

The reference composition preserves the following invariants end-to-end (each is
exercised by a runnable adversarial test in the integration harness):

1. No decision executes without a valid **kernel signature**.
2. Only the kernel mints capability tokens; no actor can forge one.
3. Advisory components (risk/threat enrichment) **cannot override a verdict** —
   they cannot even construct a decision object.
4. The enforcement/execution layer **never constructs a decision**.
5. Every routed decision yields **exactly one** valid, hash-chained audit entry.
6. Every capability token is **one-time**; replay is rejected.
7. A decision is **bound to the action's content**; a token cannot be replayed
   against a different action.
8. **Legitimacy denial is final** — the authority kernel is not consulted and
   cannot reverse it.

---

## 6. Implementation

The system is realized as a set of small, independently testable components
(Python reference; a Rust trusted computing base for the kernel with Lean 4, TLA+,
and Kani checks). The distilled public reference (`decision-os-min`) implements
the whole security model — single signing authority, content-bound decision,
one-time token, hash-chained log — in roughly 400 lines using only the standard
library plus a cryptography dependency. A cross-repo integration harness installs
the real components (not re-implementations) and asserts the invariants of §5
survive composition.

### 6.1 Trusted core in Rust; policy in Python

Two components are "rustified" because they are the parts where formal, memory-safe
verification pays off — the *trusted core* — and both are linked into the OS
distributions rather than reimplemented per version:

- **Authority TCB** — the AuthGate kernel is a Rust crate (`authgate-kernel`) with
  Lean 4 theorems, TLA+/TLC model checking, and Kani proofs. This is the authority
  decision's trusted computing base.
- **Legitimacy-kernel parity** — the FDK ships a Rust parity port of its frozen
  minimal kernel (`freedom-decision-kernel/rust/`) for cross-checking the Python
  reference against a second implementation.

The rest of the *legitimacy policy* — including the Theory-of-Freedom instance —
stays in Python: it is policy, not TCB, its risk is logic rather than memory
safety, and its failure mode is contained by the authority backstop (a wrong
legitimacy result still has to clear the authority verdict). A full Rust rewrite of
the policy layer is therefore not warranted.

### 6.2 Minimal vs. complete distributions

The same trusted core underlies two OS distributions. The **minimal** distribution
(`decision-os-min`) collapses the security model into a single ~400-line package
for a single-authority reference; the **complete** distribution composes the
separate component repos (kernel, control-plane, audit-ledger, contracts, gate)
via the integration harness. Both **link the same two rustified trusted
components** above rather than carrying divergent copies — the minimal version for
a compact embedding, the complete version for the full multi-repo deployment.

---

## 7. Evaluation

What is demonstrated:

- **Composition safety.** The eight invariants hold under adversarial logic in the
  integration harness; each has at least one runnable exploit attempt that must
  fail or be contained.
- **Determinism.** Given the same action and policy, the decision is reproducible;
  there is no learned component in the authority path.
- **Auditability.** Every decision produces exactly one tamper-evident record,
  written before the side effect.

What is **not** demonstrated (stated plainly):

- **No measured comparison.** We have not shown this beats OPA/Cedar for
  authorization, nor RLHF/Constitutional AI for safety, on any shared workload.
  The "constraint beats learned safety" hypothesis is *falsifiable and untested*.
- **Declared vs. detected facts.** Ownership, consent, and provenance are *declared
  inputs*, not detected from behavior. Detecting real coercion/deception/ownership
  is unsolved and is the system's trust boundary.
- **Scale, latency, deployment.** The harness proves security-property composition,
  not load, distribution, operational complexity, or developer misuse at scale.
- **First-party validation only.** Results are self-authored and self-tested; no
  independent evaluation.

---

## 8. Limitations and open problems

1. **The scientific claim is open.** Whether deterministic legitimacy+authority
   constraint outperforms learned alignment on a realistic agent workload is the
   actual research question and remains unanswered.
2. **The attestation gap.** The system enforces on declared facts; turning
   declared facts into verified ones is future work.
3. **Adoption, not architecture, is the gate.** There are no production users yet;
   the honest next step is deployment and validation, not more components.

---

## 9. Related work

- **Policy engines** (Open Policy Agent/Rego, AWS Cedar): decidable authorization,
  but do not separate legitimacy from authority as a structural invariant.
- **Learned safety** (RLHF, Constitutional AI): distribution-shifting, not
  guarantees; complementary rather than competing — a legitimacy policy could sit
  in front of a learned agent.
- **Capability security** (object-capability model, one-time tokens): the authority
  layer is squarely in this tradition; the contribution is composing it beneath a
  distinct legitimacy gate.
- **Deontic logic**: expresses permitted/forbidden/obligatory but leaves the terms
  free-floating; here they are attached to concrete ownership relations.
- **Formal verification of systems** (Lean/TLA+/Kani on the kernel): used to
  establish the authority TCB's properties, not the normative policy.

---

## 10. Conclusion

The problem of governing agent tool execution is not primarily a problem of making
models want the right thing; it is a problem of *authority* — who may do what, to
whose resources, recorded how. Separating legitimacy ("should this happen?") from
authority ("may this actor do it?"), and enforcing that separation structurally on
a value-neutral, auditable kernel, yields a system that fails safe by construction
and can host any normative policy on top. We have shown the architecture, its
invariants under composition, and a reference implementation — and we have been
explicit that the decisive comparative claim is not yet proven. The next work is
measurement and deployment, not more architecture.

---

## Honest status line

*A proposed architecture with a tested mechanism and an unproven thesis. The
engineering is real; the scientific superiority claim is open. This report makes
no marketing claim and should not be cited as evidence that the approach is
production-proven.*
