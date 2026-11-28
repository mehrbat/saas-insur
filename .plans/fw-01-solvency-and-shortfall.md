# FW-1 — Solvency and shortfall policy

## Context

Phase 2 G8 placeholder. The [platform overview](01-platform-overview.md#10-future-work-placeholders) assigns this work before live release. [Plan 05](05-actuarial-engine-and-simulation.md#11-aggregate-loss-premium-reserves-and-fund-depletion) provides reserve and depletion estimates. [Plan 07](07-payments-fund-ledger-and-payouts.md) provides settlement and reconciliation evidence.

APRA sets capital and insurance liability standards for authorised general insurers and Level 2 insurance groups. APRA-regulated entities within CPS 190's scope also need recovery and exit plans. Counsel and the risk bearer must confirm the applicable requirements and map them to the proposed pilot roles. Keep the project's cash triggers and simulation targets separate from regulated-entity capital and liability measures. [GPS 110](https://www.apra.gov.au/standards/gps-110) · [GPS 340](https://www.apra.gov.au/standards/gps-340) · [CPS 190](https://www.apra.gov.au/standards/cps-190)

## [P1] Prove the shortfall response before live release

- **Recommend a cash drill.** With synthetic data, reconcile settled, legally available cash and forecast the dates of approved payouts and other obligations. Test a missed contribution and a large valid claim. Exercise an early warning, a shortfall before payment falls due, and a delayed or unavailable backstop. Preserve the claim decision, amount, and due date through the escalation.
- **Assign response authority.** Finance monitors cash and owns the escalation. The risk bearer and counsel specify who owes the claim, who may authorise funding or disbursement, what the claimant must be told and when, and the fallback if cash stays short. Confirm the separate power to suspend new enrolment or begin run-off under FW-5. Define any member replenishment call in the approved terms before relying on it.
- **Defer sizing to the actuary and risk bearer.** Record the proposed cash floor, warning and escalation triggers, capital requirements where applicable, incurred but not reported (IBNR) and other claims liabilities, and whether backstop or reinsurance is needed. Any relied-on support needs an executed agreement with the provider, amount, exclusions, timing, and funding evidence. [GPS 230](https://www.apra.gov.au/standards/gps-230)
- **Coordinate dependencies.** Use FW-4 for custody and account rights, FW-7 for settlement exceptions, FW-3 for recourse, FW-5 for run-off, and FW-9 for independent model validation. Pin the Plan 05 input and result versions and the Plan 07 settlement evidence used in the drill.
- **Gate.** Before G8 and accepting live contributions, attach the signed threshold and funding policy, escalation authority matrix, and timed drill trace to the legal readiness record for the release bundle. The trace must show the correct recipient, trigger, payment status, claimant communications and lawful fallback in each case. Sign-off owners are the pilot risk bearer, actuary, finance and counsel. [Plan 05 G8](05-actuarial-engine-and-simulation.md#24-approval-and-release-gates)

## Notes

Cursor `composer-2.5` approved P1 in round 1 on 26 November 2026. The review confirmed the
G8 legal readiness record, release dependencies, and reconciliation evidence for the scoped cash
drill. The specialist solvency and funding decisions remain gated before live release.
