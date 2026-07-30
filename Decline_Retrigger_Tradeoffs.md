# Tradeoffs register — Do not send the connection again If CSP says 'I can't Install'

**Companion to** `Decline_Retrigger_Suppression_PRD.md` v1.0 (signed off 30 Jul 2026). **Not part of the PRD** — this is the PM's own record of what was decided and why, so that in six months "why does it work this way?" has an answer without archaeology.

Owner: Ashish Raj · Reviewer: Dhruv · Quality OS: Akhil (approved 30 Jul 2026)

---

## Part 1 — Decisions taken between presented options

| # | Decision point | Chosen | Rejected | Why | Date |
|---|---|---|---|---|---|
| 1 | What outcome is being bought | Trust the CSP's explicit "I can't install"; use conversion as the evidence | Framing it purely as a conversion win; framing it purely as trust with no numbers | "Data suggests that when doing so does not yield in installs." The promise is trust, the proof is the 0.4% vs 2.3% split | 29 Jul |
| 2 | Which exits still retrigger to the same CSP | **P41 and P74 only** — the two live today | Any exit with a SYSTEM source (would have auto-covered the three SCHED-01 RT timers); any exit the CSP did not deliberately cause (would have included customer-driven cancels) | "Use only P41 and P74 as these are active today." Confirmed by code: all three RT flags are pinned false in production | 29 Jul |
| 3 | What happens when suppression empties the CSP pool | **Service vacuum until P75** — routing fails, connection waits 7 days, then expires | Emitting an actionable exhaustion signal for ops to assign manually; allowing a last-resort re-offer to the decliner | Chose the existing behaviour over new machinery. A last-resort re-offer would reintroduce exactly the frustration being removed, at 0.4% conversion | 29 Jul |
| 4 | Reason-class routing (capability vs customer-claim vs logistics) | **Out of scope** | Routing declines differently by reason class, per the earlier decline-intelligence Move 1 | Every explicit refusal suppresses, whatever its reason. Reason-class routing is a later layer | 29 Jul |
| 5 | What the suppression is keyed on | **(CSP, connection)** | (CSP, customer/address), which would survive re-booking | Cheap, uses the grain that already exists. The re-booking leak is accepted and stated as a known limit | 29 Jul |
| 6 | Quality OS timeout-parity | **A stated dependency of this PRD**, with Akhil consulted and MQ-2 watching the exit mix | Leaving it out of scope entirely as Quality OS's problem | Costs one row and one measurement question; without it a scoring rule outside this spec could cancel the incentive it creates | 29 Jul |
| 7 | Refusal and timeout racing on the same allocation | **First commit wins** | Explicit CSP action always wins, even retroactively — *this was the recommendation, and it was overridden* | Chose the simpler, predictable rule. Consequence accepted: G1 carries a documented exception, and a CSP who taps Decline a second too late can still get the booking back once | 29 Jul |
| 8 | Telling the CSP about the promise | **No surface at all** — behaviour only | A confirmation at the moment of decline ("this connection won't be sent to you again"), using the already-coded but disabled CSP-app Updates/CleverTap rail; an ops/audit-only view | Ship the behaviour, not the messaging. Consequence accepted: trust accrues only as CSPs notice refused jobs stop returning | 29 Jul |
| 9 | Measuring how often the race in #7 fires | **Not measured** | A dedicated measurement question counting race-caused re-offers | Trimmed. M1's zero target is therefore verified by case inspection, not by the count alone | 29 Jul |
| 10 | How many success metrics | **Two** — re-offers after an explicit refusal (target 0) and same-CSP retries attributable to P41/P74 | Declared-refusal share of exits; install conversion of rerouted-after-refusal connections; pool-exhaustion volume | Kept only what the spec is accountable for. The other three were interesting, not load-bearing | 30 Jul |
| 11 | Whether the PRD defines states | **No state transition table at all** | Keeping the 12-row lifecycle table the template treats as canon | "A PRD should not define what the states are." The table had also invented six state names existing in no service, which risked reading as prescribing a state machine to D&A | 30 Jul |
| 12 | Whether the PRD defines configuration | **None** — no C-ids at all; §5 lists only existing parameters it relies on and does not own | A suppression trigger list, a scope parameter, an effectiveness window and a timeout list (four C-ids); before that, presenting P195/P51 as C-ids this PRD defines | Two steps. First: "if there exists configurability already, do not define it as a new requirement." Then: drop the four new ones too — the triggers are named in R1 and R2, so a list adds a moving part without adding a decision | 30 Jul |
| 13 | Whether the reason a CSP gives can exempt him | **No — suppression turns on the actor, never the reason** | Treating a CSP-submitted failure with reason `CUSTOMER_REFUSED` as "not his refusal" (which is what the draft did) | "He could game the system. The reasons provided by CSP can't be trusted on face value." Became guardrail G5 | 30 Jul |
| 14 | Can operations manually assign past a suppression | **No** | An emergency/ops override for connections visibly stuck with no eligible CSP | G1 holds "by any route". AC-SUP-9 tests the tempting case | 30 Jul |
| 15 | The trust boundary on the CSP-submitted-versus-platform-raised distinction | **Left to engineering** | Stating it as an obligation: the platform must establish it, never take a value the CSP's app asserts | Which value carries the distinction is a mechanism, and mechanisms are tech's call. Residual risk: if the label is derived from something the app controls, the guardrail is opt-out | 30 Jul |
| 16 | What happens if a suppression cannot be recorded | **Left to engineering** — no rule, no criterion | A fail-open rule (accept the refusal, apply cooldown, raise for ops, never block on an unrecorded suppression) plus its criterion | Both were carried over from the deleted state table rather than asked for. Override records the residual risk: defaulting to *suppressed* on a silent write failure locks a CSP out of a job he never refused | 30 Jul |
| 17 | Measuring G3 and G5 | **Not measured** | A measurement question recording, per suppression, whether the CSP submitted the exit and which reason he gave | Trimmed. Both guardrails still hold by construction and by their criteria; a regression would surface as CSPs quietly losing work rather than on a report | 30 Jul |
| 18 | The document's title | "Do not send the connection again If CSP says 'I can't Install'" | "Decline & Retrigger — respecting an explicit CSP refusal" | The title should state the promise, not name the mechanism, and should make clear it covers both decline and reported installation failure | 30 Jul |

---

## Part 2 — Factual corrections the PM made to the draft

Not tradeoffs, but recorded because the spec's rules and worked examples rest on them, and three of the five parameter values in the PRD contradict the locked OS documents.

| # | Claim as drafted | Corrected to | Consequence |
|---|---|---|---|
| C1 | P51 assignment cooldown = 24 hours (taken from `Demand_Allocation_OS_v1_9_1_LOCKED.md:566` and `SPR_v1_21_LOCKED.md:174`) | **4 hours** | Strengthened the case: the defect is same-day, not next-day — a CSP can refuse in the morning and be re-offered that afternoon. Recomputed the arithmetic in four criteria |
| C2 | P41 acceptance window = 2 hours (repo default and locked OS) | **6 hours** | AC-TMO-1's timeline recomputed (10:00 → 16:00) |
| C3 | P74 install grace = 72 hours (repo default) | **96 hours** | AC-TMO-3 recomputed |
| C4 | The CSP proposes an installation slot | **The customer chooses the slot at booking; the CSP either assigns a technician or declines** | Confirmed in code — acceptance is `acceptOnTechnicianAssignment`, logged as the "customer-scheduled anchor", with slot-proposal marked legacy. Removed slot-proposal framing throughout |
| C5 | "53% of re-offer chains include the decliner" | **53.1% is the second-attempt decline rate** for the 228 pairs re-offered after a decline — a different quantity | Replaced a misread proxy with the real cohort table, which is now reproduced in §1 as the empirical case for the whole spec |
| C6 | Quality OS owner is Vaibhav | **Akhil** | Came from a stale memory note of mine. Corrected in the PRD and in my notes, with a rule never to name an owner from memory alone |
| C7 | A CSP-reported failure with reason `CUSTOMER_REFUSED` is not the CSP refusing | **It is his refusal** — see decision 13 | The most consequential correction: the draft had a gaming loophole |
| C8 | Multi-account CSPs could bypass a per-CSP key | **Operator and CSP are 1-1** | Not a gap. Recorded as the assumption that makes a per-CSP key sufficient |

---

## Part 3 — Accepted residual risks

Each is recorded as an Override in the PRD. Listed together here because they are what a future reader will most want to find.

| Risk | Why it was accepted |
|---|---|
| A refusal committing just after a timeout reroute does not suppress, and its frequency is not measured | Decisions 7 and 9 |
| Suppression dies with the connection, so re-booking resets the promise for that address | Decision 5 |
| The CSP is never told the promise exists | Decision 8 |
| G3 and G5 have no post-launch measurement | Decision 17 |
| No rule for a failed suppression write; no failure-window criterion | Decision 16 |
| The trust boundary on the actor distinction is unstated | Decision 15 |
| A suppressed-and-exhausted connection dies silently at P75 with nobody notified | Decision 3 |
