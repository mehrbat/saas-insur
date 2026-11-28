# FW-7 — Payment exceptions and reconciliation

## Context

Phase 2 G8 placeholder for real-money collection and payout. [Plan 01](01-platform-overview.md)
§7.10 and FW-7 reserve exception handling. [Plan 02](02-domain-model.md) §§9.5–9.10 reserves
the live transitions and U-DM-17. [Plan 07](07-payments-fund-ledger-and-payouts.md) defines the
journal and the synthetic settlement path. FW-1 owns shortfalls, FW-3 member complaints and
FW-4 custody and payment authority.

## [P1] Rehearse payment failure and recovery before moving live funds

- **Collect and preserve rights.** Rehearse a missed first payment and a missed renewal or
  instalment. Record the payment exception separately from cover and claim decisions. A failed
  or reversed payment alone cannot suspend cover, change past cover, or undo an approved claim.
  The risk bearer and counsel define initial activation, grace, notices, suspension and lapse,
  refund eligibility, effective dates, and treatment of claims during each period.
- **Settle and reconcile.** Rehearse an unknown payout after a timeout, confirmed failure,
  a return after settlement, and duplicate or out-of-order notifications. Check the provider
  and bank before retrying an unknown outcome. Use the same idempotency key for an exact retry
  within the provider's guarantee. A replacement attempt requires confirmed finality of the
  prior attempt and, when required, fresh approval. Preserve the original claim decision,
  approved amount and unpaid liability until verified settlement or an authorised disposition.
- **Correct the journal.** Rehearse a refund, reversal and chargeback, including a return that
  arrives after the initial settlement reconciled. Post balanced, idempotent compensating
  entries linked to the original posting and provider or bank evidence. Keep an exception open
  until the cash, obligation and provider records reconcile or an authorised owner resolves
  the discrepancy. Require distinct maker and checker identities for adjustments or replacement
  disbursements when the signed authority matrix requires them. Escalate cash gaps to FW-1.
- **Gate.** Before G8, finance and engineering pin the selected provider's authenticated event,
  retry and settlement contracts and supply replayable fault-drill traces. The risk bearer and
  counsel sign the cover, notice, complaints and approval rules. Each trace must show the final
  cash and liability balances, reconciliation outcome, member notice where applicable, approval
  evidence, and zero duplicate payment or journal postings. Leave provider selection and live
  policy rules open until those owners sign off.

## Notes

Cursor `composer-2.5` approved P1 in round 1 on 26 November 2026. The review
confirmed the G8 gate, the reserved payment and cover transitions in Plan 02,
and the reconciliation and idempotency requirements in Plan 07. Detailed live
policy and provider selection remain open. Plan 07 requires partial-failure
reconciliation before enabling any permitted fee transfer.
