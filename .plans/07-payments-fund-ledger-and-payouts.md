# Plan 07 — Payments, fund ledger, and payouts

## Context

Draft for Phase 1 synthetic money and the G7 test release. The first fixture is the funded
fixed-term event from Plan 03. Keep Plan 02's subscription fixture at the second conformance
milestone. This plan defines posting, settlement and reconciliation. Plan 04 owns the human
decision, Plan 05 owns the final amount and trace, and Plan 02 owns the domain state machines.

Cursor `composer-2.5` approved P2 and P3 in round 1 and P1 and P4 in round 2 on
26 November 2026. The revisions use the existing per-ledger S-6 constraint for the derived
journal key and identify the fixed-term and subscription acceptance milestones separately.
Plan 05 §12.5 now names the engine-only realm for AT-SIM-1.

References

- [Plan 01](01-platform-overview.md) §§7.8–7.10, 7.12, 7.16 and FW-1, FW-4, FW-7.
- [Plan 02](02-domain-model.md) §§6.6, 8, 9.5–9.7, 10–12, 14.2, 14.5, 15 and 20.
- [Plan 03](03-architecture-and-tech-stack.md) §§3–4, 8–9.
- [Plan 05](05-actuarial-engine-and-simulation.md) §§9, 11.4, 12.5, 19.4, 24, 26 and AT-SIM-1.

## [P1] Define the journal and posting rules

Implement the money command handlers and account recipes in the Plan 02 Python domain core,
and persist them through Plan 03's PostgreSQL aggregate adapter. The `Ledger` aggregate owns
the chart and append-only entries. Create one fund and one fund ledger per realm. Read the
scope, currency, `money_kind`, and immutable `ledger_source` of the realm from storage. Only
the Plan 07 service principal may call `PostJournalEntry`, `RecordContributionPaid`,
`RecordPayoutSent`, or `RecordPayoutSettled`. A command names its immutable cause and target.
The posting handler derives account lines from the cause and recipe. It accepts neither an
operator-supplied journal nor a general-purpose cash transfer.

All amounts are positive integer minor units on a line. Direction comes from the debit or
credit side. Each entry has `ledger_id`, business `entry_date`, `intraday_order_key`, `tie_key`,
`cause_ref`, `idempotency_key`, typed lines and an optional `transfer_id` for reserved live fee
transfers. Each line references an account in the same ledger and currency. Reject invalid
amounts, currency, recipes, scope, causes or unbalanced entries before committing. A ledger
write commits the entry, aggregate version, audit, outbox and command result atomically. Use the
existing S-6 unique constraint on `(ledger_id, idempotency_key)`. The handler derives the journal
`idempotency_key` as the canonical JSON of `[cause_kind, cause_identity, recipe_code]` under
Plan 02 §5.6, with the recipe fixed by the validated cause. Use the due ID, payout ID, or
`fee_charge_ref` for domain facts and `[artifact_id, event_id]` for engine events. The command key is independent of this
journal key, so changing the command key cannot double-post a cause. Plan 02 §11.1 exact command
retries return the stored result. A different command that derives the existing journal key
receives a duplicate error.

Keep the Plan 02 fund chart and add these fixture posting recipes. Keep `contributions_receivable`,
`claims_payable` and `refunds_payable` as reserved accounts pending accrual policy. In Phase 1,
outstanding obligations are read from domain or model records. The journal records cash effects.

| Cause | Debit | Credit |
|---|---|---|
| Immutable paid contribution | `fund_cash` (asset) | `member_equity` (equity) |
| Settled claim payout | `claim_payout_expense` (expense) | `fund_cash` |
| Engine refund event | `member_equity` | `fund_cash` |
| Engine `other` outflow | `model_outflow_expense` (expense) | `fund_cash` |
| Named synthetic fee from contributions | `simulated_fee_sink` (expense) | `fund_cash` |

For the Phase 1 default `fee_treatment = separate`, leave the fund and dummy fee sink untouched.
One conformance fixture uses `from_contributions` on a `domain_flow` run. It posts a gross paid
contribution and a published fixture fee draw to the dummy sink on the next business day. Pin
`FeeCharge` to the fund, period, `fee_schedule_ref`, and fixed amount. Accept one fee draw per
`fee_charge_ref`, regardless of retry key, and reject any claim-linked or outcome-derived charge. Create
`service_fee_payable` only for `from_contributions`; the cash fixture does not use the payable
account. No simulation realm has a platform ledger, custody reference, external rail credentials
or real counterparty. Record actuarial reserves as reported estimates. Defer booked reserves,
accrual, actual refunds and live platform transfers to their release gates below.

## [P2] Settle the end-to-end dummy path exactly once

On a `domain_flow` realm, use approved fixtures to schedule the synthetic payment facts and use
engine generator output only to seed domain claims and dues. `RecordContributionPaid` binds a
same-scope due in state `due` to an immutable synthetic `payment_ref` and its exact due amount.
Every paid due causes one contribution journal entry. The first paid due also triggers the
Plan 02 cover activation check.
The same due cannot acquire another payment reference. A malformed or conflicting fact stops
the worker and is visible in the run audit.

Only the process runner may `CreatePayout`. It uses a payable claim, its executed committee
approval, and the positive committed final amount and trace recorded by Plan 02 §14.2. The
payee comes from the claim domain record. `claim_decision_id` is the unique payout key. No
payout is created for rejections, zero amounts, waiting cohorts, previews, or AI opinions.
The Plan 07 service advances `instructed → sent → settled` on the realm clock, using a stable
synthetic `rail_ref` bound to the payout and its exact amount, currency and payee. Use Plan 02
§15.2 step keys and same-instant rescans for a payout created by a cohort close. The live
provider interface separately rejects dummy money at its boundary. Reject live payment and
payout execution in the Phase 1 deployment.

One `PayoutSettled` event causes one cash journal. Use separate aggregate commands for sent,
settled, journal and Plan 02 claim closure. Never hold multiple aggregate locks. Deliver journal
events with the transactional outbox and stable operation keys. On a lost response, return the
recorded result or read the authoritative entry for that cause before acknowledging the outbox.
Derive each journal's business date from the immutable fact in the realm time zone. Use the
fixture's pinned intraday order and stable cause IDs to sort the cash history independently of
delivery order.
Compare settled payouts, paid dues and due fixture fee facts to journal causes at the same realm
watermark. Flag missing, excess and amount-mismatched entries. A settled payout or closed claim
may precede its journal. Show that lag as missing in reconciliation until the entry commits. A
dummy report and G6 completion require an empty unmatched set through the pinned watermark and
an exact match to the fixture's expected cash total.
The funded fixture checks expected cash after every posting. A depleted domain fixture fails
conformance and is held for FW-1. Engine projections may show negative simulated balances.

## [P3] Replay the engine path and reconcile every balance

In an `engine_cash_flows` realm, validate and bind the immutable Plan 05 cash-flow artifact,
calculation/run reference, seed, currency and starting balance. The realm has no actual domain
claims, dues or payouts. Event member and claim IDs are provenance labels. Reject the wrong
post source in both realms before a journal write. Use one replay manifest per realm. Refuse a
second manifest or changed payload under the same artifact ID.

Use the same cash recipes on `contribution`, `claim_payment`, `refund`, and supported `other`
outflow events. The contribution amount is the amount received by the fund per Plan 05 §11.4.
Do not deduct a synthetic fee again from those receipts. The engine owns the net amount and
other outflows. Apply the published tuple `(date, intraday_order_key, tie_key)` and retain it
unchanged. Validate unique event IDs, unique order tuples, permitted types and nonnegative minor
unit amounts before posting. A bad event fails the batch with its ID and a reason. Zero-amount
events produce a no-post replay position with identical before and after cash. Positive events
post with the pinned artifact and event ID as provenance and stable identity. Reconstruct every
prefix balance from the ordered journal and zero-event positions. Process one event at a time
through the Plan 03 §7.2 replay job. Advance its fenced checkpoint only after reading the matching
journal or computing the zero-event receipt. On a crash between journal commit and checkpoint,
verify the committed entry and advance without reposting. On resume, compare every prefix with
the pinned engine path. Fail at the first mismatch with its event, expected cash and actual cash.

For the first AT-SIM-1 fixture pin `starting_fund_balance = 0`. Support a nonzero opening balance
only after Plans 02 and 05 agree an engine-authorized opening event preceding all ordinary flows.
Recommended shape is a distinct opening event from the same immutable artifact, posting debit
`fund_cash`, credit `member_equity` once, with a separately pinned amount equal to `B0`. Until the
producer and contract accept this shape, reject `B0 != 0` for ledger replay. Never invent a
balancing adjustment to hide a mismatch. Plan 02 §14.5 is the acceptance scope for AT-SIM-1.
Request that Plan 05 §12.5 name the engine-only realm explicitly. The committee flow has a
separate fixture with independently expected contributions and settled payouts.

## [P4] Verify Phase 1 and track live release gates

| Acceptance case | Required evidence |
|---|---|
| L1a - fixed-term contribution and payout (A3) | First due paid, cover activated, executed partial approval yields a lower positive final amount, payout sent and settled at fixture time, claim closed. With a 10,000 contribution, 4,500 committee approval and a 4,000 final payout, cash is 0, 10,000 then 6,000 minor units. Assert cash and audit at every step. |
| L1b - subscription renewal (A4) | The second fixture exercises subscription dues across renewal. Assert its due schedule and reconcile posted contributions and cash through the renewal watermark. |
| L2 — authority and isolation | Rejected, zero, pending and preview claims produce no journal payout. Wrong tenant, realm, currency, cause, principal, ledger or source is refused. The deployment cannot call live rails, and an adapter presented with a dummy scope refuses independently. |
| L3 - duplicates and recovery | Concurrent `CreatePayout` commands bind once. Re-deliver sent, settled and paid facts, kill the worker after settlement and after journal commit, and resume a clock advance at its target instant. Assert the same payout, journal and cash. Submit a second command key for the same paid due, settled payout and fixture fee. Each derives its original journal key and is refused without a second posting. |
| L4 — ledger invariants | Run Plan 02 AT-DM-MON-1–7, with the Plan 02 live fund-to-platform case reserved until live adapters exist and the dummy fee case verified now. An amount, currency or source mismatch fails without changing cash. Equal retry returns the original S-1 result. |
| L5 — AT-SIM-1 | Fixed seed, same-day mixed flows, zero event and negative intermediate balance followed by recovery. Every balance agrees at the same event. Reordered or duplicate events exercise deterministic ordering and validation. A wrong balance fails before completion. |
| L6 — G6 and G7 | Report includes the pinned ledger watermark, reconciliation result, cash totals, test run IDs and selected fixture evidence. G6 stays pending while an expected posting is missing. G7 runs with the same signed bundle under Plan 02 §15.4 and Plan 03 §8. |

Deliver L1a and L2–L4 with the fixed-term fixture on Plan 03 A3. Replace stub adapters and complete
L1b, L5 and L6 with the subscription and engine fixtures for A4. Live adapters and G8 remain
subject to the release dependencies below.

| Placeholder | Release dependency and next decision |
|---|---|
| FW-1 — shortfall and solvency | Legal, actuarial and operations owners choose the cash and capital thresholds, shortfall path and authority to defer or fund a valid payout. Prove the agreed live control before accepting real contributions. |
| FW-4 — custody and provider contract | Legal and finance identify issuer, custody title and member rights, provider account ownership, settlement authority and reconciliation obligations. Only then design live collection and payout adapters and their trust boundaries. |
| FW-7 — payment exceptions and reconciliation | Finance and engineering design provider event authentication, external idempotency, settlement/return reconciliation, chargebacks, refunds, reversals, grace periods, paid-cover effects and second approvals. Prove end-to-end recovery before live funds move. |
| Live finance — fees, accounting and access | Resolve U-DM-4 and U-8. Legal and finance approve the service fee contract, accounting basis, booked reserves, record retention, ledger access, separate platform accounts and what counts as fund cash. A permitted from-contributions transfer uses one fee reference and paired, independently balanced journal entries linked by `transfer_id`. Define partial-failure reconciliation in FW-7 before enabling it. |
