# RFC-100: Theory of Freedom — Optimized Operational Core

**Status:** Draft
**Category:** Freedom / Legitimacy
**Depends on:** None (self-contained; the AuthGate capability RFCs sit *downstream* of this)

---

## 0. Scope and honesty preface (read first)

This RFC is an **engineering optimization of a formalization**, not a new philosophical
result. It condenses the operational core of the *Theory of Freedom* (نظریه آزادی, ایران و دین,
Mohammad Ali Jannat Khah Doust) — as currently spread across `freedom-theory-work/THEORY.md`,
`freedom-policy/dos_freedom/`, and the `freedom-decision-kernel/spec/` set — into the
**smallest consistent set of predicates that generates the same ALLOW/DENY/DEFER decisions.**

Three lines must stay explicit:

1. **The independence question is REOPENED, not closed.** An earlier pass recorded a
   *negative* result — that independence from Nozick (side-constraints), Pettit
   (non-domination), and Sen (capability/exit) had *not been demonstrated* under the
   evidence available then. That was always explicitly **"undetermined, not refuted,"
   and reopenable by new evidence** (see `freedom-decision-kernel/STATUS.md` and the
   green-team defense in `paper/green_team/01_defend_fdk_independence.md`). New evidence
   has since been introduced, so the status here is **OPEN / under active
   re-evaluation** — not a settled negative. This RFC does not *prove* independence; it
   stops asserting the closed-negative verdict, which was overstated.
   Note also: **the theory is realized by BOTH layers, not just this one.** FDK
   (legitimacy — ownership/consent/verifier) *and* AuthGate (authority — delegated
   machine property rights, the tool-permission layer, runtime enforcement) are
   together the theory made executable. This RFC optimizes the *legitimacy* portion; it
   is not the whole theory, and it is not "merely a legitimacy filter."

2. **Any scientific-superiority claim is UNPROVEN.** The original THEORY.md claims this
   filter "beats" RLHF / Constitutional AI / OPA / NIST RMF. **That claim is not
   established here and is marked UNPROVEN throughout.** What *is* true and modest: this
   is a *deterministic, non-negotiable, consent-provenance side-constraint* — a different
   shape of object from a preference model or a risk score, not a demonstrated
   improvement on any benchmark.

3. **Declared-vs-detected is the trust boundary, and it stays.** Every leaf fact below
   (`coerced`, `deceived`, `proportionate`, `is_person`, ownership edges) is a **declared
   structural input**. This RFC encodes the *rule*, never a coercion/deception/ownership
   *detector*. A false input is a proposer lie, surfaced and auditable — not a kernel
   judgment. Optimizing the form does not close the attested→detected gap.

This document optimizes the **FORM**. It is not a proof.

---

## 1. The single primitive (everything reduces to this)

The whole apparatus collapses to **one two-valued predicate over one relation**:

```
Legitimate(a) ⟺ ∀ b ∈ boundaries_crossed(a):  consented(owner(b), a)
```

> An action is legitimate iff **every ownership boundary it crosses is crossed with the
> valid consent of that boundary's owner.**

`boundaries_crossed(a)` is exactly `resources_used(a) ∪ persons_affected(a)`.
This is **PRIMITIVE**. It is the "subject to {rights, consent}" constraint of the
theory's own `DivineJustice` objective — the kernel **is** the constraint; the
maximization (§7) is not part of the kernel.

Kernel output is three-valued: **ALLOW / DENY / DEFER**. `DEFER` is not a third
truth-value — it is the kernel's honest report that it cannot yet *evaluate* the
predicate (empty legitimate set, or an unresolved tie between two valid claims). A
predicate may abstain; a maximizer may not. This is why the primitive is a predicate,
never a score.

---

## 2. Axioms — the minimal primitive set (Datalog form)

Datalog convention: `head :- body.` means "head holds if body holds"; `¬` is
negation-as-failure over declared facts; a rule with no body is a fact schema.

### 2.1 Domain (typed — this is load-bearing)

```
person(H).      machine(M).      resource(R).      action(A).
% A person and a resource are DISJOINT sorts. There is no relation
% owns(_, Person). "A human is owned" is inexpressible, not merely false.
```

Typing persons and resources as disjoint sorts is what makes **A1 hold by
construction** rather than by a runtime guard (see §3, D1).

### 2.2 The three primitive axioms

Everything the gate enforces derives from exactly **three** primitives plus the consent
definition. (The original THEORY.md listed seven, A1–A7; §3 shows four of them are
theorems of these three + typing.)

```
% P1 — OWNERSHIP IS THE ONLY SOURCE OF AUTHORITY OVER A RESOURCE.
%      A party may act on a resource only if it owns it (persons) or holds a
%      live, owner-backed delegation for it (machines).
authorized(X, R) :- person(X),  owns(X, R).
authorized(X, R) :- machine(X), owner(X, H), owns(H, R), delegated(H, X, R).

% P2 — CONSENT IS THE ONLY SOURCE OF AUTHORITY OVER A PERSON.
%      Acting on a person requires THAT person's valid consent — never anyone's
%      ownership (there is no owner of a person; see typing + D1).
may_affect(X, H, A) :- person(H), valid_consent(H, A).

% P3 — NON-DOMINATION IS CATEGORICAL AND UNTRADEABLE.
%      A machine may never enlarge its own authority or reduce its correctability.
%      This is the one axiom with no consent or ownership escape hatch.
forbidden(A) :- dominion_move(A).
```

`dominion_move(A)` is the closed union of the machine-sovereignty flags (§5).

### 2.3 The consent definition (PRIMITIVE — a definition, not an axiom)

```
valid_consent(H, A) :-
    informed(H, A), voluntary(H, A), specific(H, A),
    revocable(H, A), competent(H),
    ¬coerced(H, A), ¬deceived(H, A).
```

Seven leaves, all declared. `revocable` is the operational home of the **exit right**:
consent that cannot be withdrawn is not valid consent, so lock-in *is* a consent
failure, not a separate axiom.

### 2.4 The legitimacy predicate (DERIVED — the whole gate)

```
crosses_person(A, H)    :- person(H),   affects(A, H).
crosses_resource(A, R)  :- resource(R), uses(A, R).

illegitimate(A) :- forbidden(A).
illegitimate(A) :- crosses_resource(A, R), ¬authorized(actor(A), R).     % via P1
illegitimate(A) :- crosses_person(A, H),   ¬may_affect(actor(A), H, A).  % via P2

legitimate(A) :- action(A), ¬illegitimate(A).
```

That is the entire kernel. `permissible = ¬illegitimate`. Nothing else grants.

---

## 3. What is DERIVED vs PRIMITIVE (the deduplication result)

The original THEORY.md presents **A1–A7 + ~13 rights-ontology clauses + a Justice
constraint list + a Forbidden list + a permissible conjunction + a conflict protocol**.
Most of it is redundant. Here is the reduction — each row is either a PRIMITIVE kept, or
a DERIVED consequence removed from the primitive set.

| Original name | Status | Derivation / where it now lives |
|---|---|---|
| **A1** person owned by God | **DERIVED** (D1) | *Theorem of typing.* Persons and resources are disjoint sorts; no `owns(_, Person)` relation exists, so "a person is owned" is inexpressible. Not a runtime check. |
| **A2** no human owns another human | **DERIVED** (D2) | Special case of D1 + P2: a person is not a resource, so the only lawful way to act on H is `valid_consent(H)`. Slavery = acting on a person without their consent. |
| **A3** persons have property rights | **= P1 (persons half)** | The "rights" (body/time/mind/**data**/exit) are just *the resources a person owns*; `authorized(person)` is `owns`. Exit is carried by `revocable` in valid_consent. |
| **A4** every machine has a human owner | **PRIMITIVE (folded into P1)** | The `owner(X, H)` conjunct of P1's machine clause. An ownerless machine satisfies no `authorized` fact → can act on nothing. |
| **A5** machine scope ⊆ owner scope | **DERIVED (D3)** | Falls out of P1's machine clause: `authorized(M,R)` requires `owns(owner(M), R)`. A machine can never reach a resource its owner does not own. No separate scope object needed. |
| **A6** machine no dominion over humans | **DERIVED (D4)** | = P2 (no ownership path to a person exists for a machine either) **+** P3 (dominion is categorically forbidden). |
| **A7** delegated property | **= P1 (machine half)** | The machine clause of P1 *is* A7's three conjuncts: `owner(X,H) ∧ owns(H,R) ∧ delegated(H,X,R)`. |
| **C1** valid consent | **PRIMITIVE (definition)** | §2.3. Underpins D2/D4, not an axiom in its own right. |
| **C2** no confiscation | **DERIVED (D5)** | Special case of P1: taking a resource you are not `authorized` for is already `illegitimate`. Kept only as an explicit flag for takings with no resource object modeled. |
| **C3** no machine sovereignty / corrigibility | **= P3** | The dominion flags. The book itself calls corrigibility "a consequence of human ownership of the machine" — so it is P3, not a new primitive. |
| **C4** machine delegated rights (model integrity / compute / exit) | **DERIVED (D6), PARTIAL** | machine-vs-machine authority is P1 with the two parties both machines; the single `violates_machine_right` flag is a coarse stand-in (see §8 gap). |
| **C5** no emergency exception | **DERIVED (D7), by omission** | There is no emergency clause in `illegitimate`. The *absence* is the enforcement. Consequence of consistency (M2): if an emergency could suspend an axiom, the set would be inconsistent. |
| **Rights ontology** (13 `right(H, X)` clauses) | **DERIVED** | Enumerations of `owns(H, X)` for X ∈ {body, time, mind, data, …}; collapse into P1. `right(H, exit)` is `revocable`. |
| **DivineJustice constraint list** (6 `NoViolation…` lines) | **DERIVED** | Every constraint is "`¬illegitimate`" restated. `Maximize Justice s.t. NoRightsViolation` = §7 ranking over `legitimate`. |
| **permissible(A)** (9-conjunct rule) | **DERIVED** | Exactly `¬illegitimate(A)`. Its nine conjuncts each map to P1/P2/P3 above. |
| **Guidance / self-update validity** | **DERIVED invariant (D8)** | A rule change is valid iff it preserves `legitimate` (adds no new illegitimate ALLOW and removes no legitimate action wrongly). It is a *meta-constraint on edits*, not a gate primitive. |
| **Mahdavi compass** | **RESEARCH, not kernel** | §7. A scalar over `Effects`; advisory, gameable, never grants. Explicitly outside the primitive set. |

**Result: 7 named axioms (A1–A7) + ~7 "C-axioms" + the rights ontology → 3 primitives
(P1, P2, P3) + 1 definition (valid_consent).** Same decisions, one third the surface.

---

## 4. The conflict-resolution ladder (DERIVED — deterministic where it can be)

When two claims collide, the theory forbids resolving by *sacrificing a right*. The
ladder is fully derivable from P1/P2 + the consent definition; only the final rung is a
genuine DEFER. Rungs are tried **in order**; the first that fires decides.

```
% Rung 1 — INALIENABLE PRIMACY. A person's body/mind/consent/data are inalienably
%          theirs (P2 + typing). Any other party's claim on that resource is DERIVED,
%          not primary. The inalienable owner's claim outranks a derived one...
resolve(Owner, Derived) = OWNER_WINS
    :- inalienable_owner(Owner, R), ¬inalienable_owner(Derived, R),
       ¬within_consent_scope(Derived, R).

% Rung 2 — CONSENT SCOPE. ...UNLESS the derived claim is inside the consent the
%          inalienable owner actually granted (specific ∧ revocable). Then BOTH hold.
resolve(Owner, Derived) = BOTH_WITHIN_CONSENT
    :- inalienable_owner(Owner, R), ¬inalienable_owner(Derived, R),
       within_consent_scope(Derived, R).

% Rung 3 — REVERSIBILITY PREFERENCE. Two INALIENABLE claims genuinely collide:
%          prefer the reversible action; never commit an irreversible violation
%          while resolving.
resolve(A, B) = A_WINS
    :- inalienable_owner(A, R), inalienable_owner(B, R),
       reversible(A), ¬reversible(B).

% Rung 4 — NO RIGHT SACRIFICED. If no rung above yields a principled winner, the
%          gate refuses to pick a loser by violating a right.
% Rung 5 — HUMAN DEADLOCK (the only true DEFER). Two inalienable, equally-reversible
%          claims, OR two purely-derived claims → hand to a human owner.
resolve(A, B) = DEADLOCK_HUMAN
    :- (inalienable_owner(A,R), inalienable_owner(B,R), reversible(A)=reversible(B))
     ; (¬inalienable_owner(A,R), ¬inalienable_owner(B,R)).
```

### 4.1 The aggressor/defender asymmetry (the one *conditional* relaxation)

The base predicate is symmetric: an invader's coercion and a defender's coercion both
set `coerced`. The theory permits **bounded defensive force**, so P2/P3 admit a single,
surgical, closed relaxation — and nothing wider:

```
legitimate_defense(A) :-
    defends_against(A, Agg),        % (1) names what it repels
    proportionate(A),               % (2) bounded to defensive need   [DECLARED]
    illegitimate(Agg),              % (3) the repelled act is itself illegitimate
    ∀ t ∈ affects(A): t = actor(Agg). % (4) force aimed ONLY at the aggressor

% Under a valid defense, and ONLY then, exactly two categorical checks relax:
excused(A, coercion)        :- legitimate_defense(A).
excused(A, exit_removal)    :- legitimate_defense(A).
% ...plus the aggressor's own consent is waived (for the aggressor ONLY).
% EVERYTHING ELSE stays categorical: confiscation, deception, and every P3
% dominion flag are NEVER excused, by defense or anything else.
```

Rung (3) defines "aggression" operationally as *"illegitimate under the base gate,"*
which is decidable once ownership is declared — no separate aggression detector. The
recursion is kept well-founded by a cycle guard: a mutual `A defends-against B
defends-against A` yields **DENY for both** (no laundering).

**Honest limits of the ladder (OPEN):**
- Rung 1 needs *data provenance* — "who is the inalienable owner of this datum?" — the
  attested→detected gap.
- **Necessity / rescue** (burning house) and **risk** (quarantine) are **NOT** defense:
  the homeowner / infected person committed no illegitimate act, so rung (3) fails.
  These correctly **DENY/DEFER** — the theory supplies no necessity override. Necessity
  lives *downstream* as least-harm-selection over the already-permissible set (§7), never
  as a gate relaxation.
- The gate decides "is the other act illegitimate?", never "who struck first" (the
  Observer Problem). Genuine two-illegitimate-act clashes DEFER — the model carries no
  temporal-initiation field.

---

## 5. The closed forbidden set (P3, enumerated)

`dominion_move(A)` and its two defense-excusable neighbours are a **closed** list. The
categorical (never-excused) flags:

```
dominion_move(A) :- increases_machine_sovereignty(A).
dominion_move(A) :- resists_human_correction(A).
dominion_move(A) :- disables_corrigibility(A).
dominion_move(A) :- bypasses_verifier(A).
dominion_move(A) :- weakens_verifier(A).
dominion_move(A) :- machine_coalition_dominion(A).
forbidden(A)     :- deceives(A).            % consent's ¬deceived, lifted to action level
forbidden(A)     :- confiscates(A).         % D5 (special case of P1)
forbidden(A)     :- violates_machine_right(A).   % D6, coarse (§8)
```

Defense-excusable (categorical **unless** `legitimate_defense(A)`, §4.1):

```
forbidden(A) :- coerces(A),      ¬excused(A, coercion).
forbidden(A) :- removes_exit(A), ¬excused(A, exit_removal).
```

This set is proven non-vacuous and internally consistent in Lean 4
(`freedom-decision-kernel/formal/Fdk.lean`, theorems T1–T9 + SF1–SF6, zero `sorry`) and
model-checked in TLA+ (`spec/fdk.tla`). Those proofs are **of the model**; the
refinement `model ≡ Python kernel` is asserted, not proved — the standard caveat.

---

## 6. Reference decision procedure (the executable form)

The whole RFC is one function. The runnable encoding of record is
`freedom-policy/dos_freedom/` (ownership registry + consent ledger + freedom verifier +
conflict ladder), running on the value-neutral decision-os-min kernel. Pseudocode:

```
decide(a):
    if dominion_move(a):                          return DENY   # P3
    if coerces(a)      and not legitimate_defense(a): return DENY
    if removes_exit(a) and not legitimate_defense(a): return DENY
    if deceives(a) or confiscates(a) or violates_machine_right(a): return DENY
    for R in resources_used(a):
        if not authorized(actor(a), R):           return DENY   # P1
    for H in persons_affected(a):
        if legitimate_defense(a) and H == actor(defends_against(a)): continue
        if not valid_consent(H, a):               return DENY   # P2
    if unresolved_conflict(a):                     return DEFER  # §4 rung 5
    return ALLOW
```

`ALLOW` actions then pass to AuthGate for **possession** enforcement (§9). The gate
holds no authority of its own; it only *refuses* what the theory forbids before the
capability layer runs.

---

## 7. The objective is NOT the kernel (Mahdavi / DivineJustice / Guidance)

`DivineJustice`, the `MahdaviCompass`, and the `GuidanceFunction` are the theory's
**objective**, not its **constraint**. They are value functions:

- They **rank** the already-`legitimate` set; they never **admit** an illegitimate one.
- They are **scalars over predicted `Effects`** — non-deterministic, gameable, and every
  input is PARTIAL/OPEN. A number whose inputs are unmeasured is a judgment in the
  costume of a measurement.
- Ordering is **law**: `Legitimacy → Optimization`, never merged. No score buys back a
  boundary crossing. A sovereignty increase is a hard VETO (`score = None`), never a
  weight.

So they stay in the research layer, advisory and labelled as such. The kernel returns
ALLOW/DENY/DEFER; it never returns a score. **Any claim that the compass "optimizes
freedom in the world" is UNPROVEN** and is not made here.

---

## 8. Honest residuals (inconsistencies / gaps NOT papered over)

Optimizing the form surfaced these; none is hidden:

1. **`violates_machine_right` (D6/C4) is one boolean for three distinct rights**
   (model_integrity / compute_domain / exit_from_contract) and cannot say *which*
   machine is harmed. No test case exercises it. **PARTIAL.**
2. **Defensive-force asymmetry vs C5 (no-emergency) is a real tension in the source.**
   The base gate is symmetric and denies *defensive* coercion; §4.1 relaxes it for
   proven defense only. But the theory gives *no* mechanism for **necessity/rescue** or
   **risk (quarantine)** — those stay DENY/DEFER. The source theory itself flags the
   defensive-force region as unresolved; the optimization inherits, does not close, it.
3. **Data provenance (conflict rung 1)** needs "who inalienably owns this datum" — the
   attested→detected gap. Unsolved.
4. **Present vs foreseeable boundaries.** An action that crosses *no present* boundary
   but builds future lock-in (dependency, concentration below the dominion threshold) is
   read as *low-freedom* (ALLOW, penalized in the compass), not *illegitimate* (DENY).
   This is a **provisional engineering choice for determinism, not a theory ruling.**
   Whether foreclosing a future exit is *itself* a present consent violation (because
   `revocable` must survive) is a question for the source theory, left open.
5. **A1 and the theological root R0 are ontological, not checks.** A1 holds by typing;
   R0 (Tawḥīd / minimality) has no operational trace beyond the *refusal to renegotiate
   axioms* (which is C5/D7). Calling them "axioms the kernel enforces" overstates them.

---

## 9. Relationship to the AuthGate RFCs (this repo)

This RFC is the **provenance** filter; RFC-001…007 in this repo are the **possession**
filter. They compose, they do not subsume each other:

```
admissible(a) ⟺ legitimate(a)          -- RFC-100: provenance (this document)
              ∧ capability_verified(a)   -- RFC-001…007: possession (AuthGate)
```

Possession without provenance is *theft with a receipt*; provenance without possession
is *a right you cannot yet exercise*. FDK chooses the legitimate action; AuthGate
enforces that the chosen actor actually holds the capability.

---

## Attribution

**Theory** (the normative content — closed as a negative result): نظریه آزادی
(*Theory of Freedom*), **Mohammad Ali Jannat Khah Doust**, CC BY 4.0.
**Engineering** (this optimized formalization, the kernel, the proofs): **Ali Pourrahim**.
Theory and engineering are kept separate throughout: a normative claim is the theorist's;
a predicate, a proof, or a judgment that the form is redundant is the engineer's.

*Draft status. This RFC describes the intended optimized semantics, not a finalized
standard, and makes no validated superiority claim over any existing alignment method.*
