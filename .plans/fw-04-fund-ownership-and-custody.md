# FW-4 — Fund ownership in law and custody

## Context

Australian Phase 2 G8 placeholder. [Plan 01](01-platform-overview.md)
D1 assigns claims decisions to the committee and payout execution to the platform. D2 gives
members collective fund ownership. [Plan 07](07-payments-fund-ledger-and-payouts.md) depends on
the approved legal and custody structure before designing live collection and payout adapters.

## [P1] Establish fund ownership and authority before G8

Product and Australian counsel should test the licensed-partner route first and document the
differences in member promises and rights across three candidate structures.

| Structure | Question to resolve |
|---|---|
| APRA-authorised member-owned insurer | Who owns the entity and the particular assets, and what enforceable rights do policyholders have to the pool and any surplus? [Insurance Act 1973 §§9–12](https://www.legislation.gov.au/C1973A00076/latest/text) |
| APRA-authorised partner insurer | Who bears the insured liability under the policy, what claims authority can the partner give the committee, and what legally distinct pool or rights could members retain? [APRA register](https://www.apra.gov.au/registers/list-registered-general-insurers) · [ASIC INFO 253](https://www.asic.gov.au/for-finance-professionals/afs-licensees/apply-for-an-afs-licence/licensing-for-certain-service-providers/claims-handling-and-settling-how-to-comply-with-your-afs-licence-obligations) |
| Discretionary mutual | What does the fund undertake to consider or pay, who has discretion, and how do member rights and recourse change? Confirm applicable regulation for the terms and activities. [APRA discretionary mutual funds](https://www.apra.gov.au/non-regulated-industry/other-non-regulated-entities) |

Require a responsibility and beneficial-rights matrix signed by Australian counsel, the risk
bearer and the custodian. For premium, member equity or contributions, refunds, surplus and
claim proceeds, record legal and beneficial title over the money's lifecycle, entity that owes
the member, bank account holder, segregation basis, controlled access and reconciliation, claims
decision authority, payment instruction and execution authority, and rights on shortfall,
insolvency or wind-down. Confirm which licensing and client-money rules apply to each activity.
[Corporations Act 2001 §§766G, 911A and Part 7.8 Divisions 2, 4 and 5](https://www.legislation.gov.au/C2004A00818/latest/text)

G8 evidence includes the signed matrix, executed agreements and account mandates, approved
member disclosures, and a walkthrough tracing one contribution, claim payout and insolvency
scenario to the documented rights and authorities. Return any conflict with D1 or D2 to the
product owner for explicit resolution before release. Hand the approved authority and funds flow
to Plan 07 and the shortfall, closure and committee work in FW-1, FW-5 and FW-6.

## Notes

Cursor `composer-2.5` approved P1 in round 1 on 26 November 2026. The verdict confirmed the
D1 and D2 decision boundaries, the required G8 evidence and the Plan 07 live adapter dependency.
The choice of legal structure and the signed evidence remain G8 requirements.
