# Plan 09 — Implementation sequence for Plans 02–08

Start with the shared contracts in Plans 02 and 03 and the artifact contracts in Plan 05. Then build the platform and standalone engine in parallel. Plans 04, 06, 07 and 08 can start against versioned fixtures as soon as their own interfaces are agreed. They join at A3 for the first integrated claim and at A4 for the shared G7 test release.

Use the synthetic fixed-term sports event as the first fixture. Keep the subscription fixture for second-type conformance. The repository currently contains plans, a README and a licence. Start with scaffolding, executable contracts and test fixtures.

## 1. Agree the interfaces first

Run Plan 03 A0 with the contract portion of Plan 02 DM0 and Plan 05 M0. Publish the schema skeleton, shared test vectors, fixture versions and ownership before parallel changes to shared interfaces.

| Interface | Owners | First handoff |
| --- | --- | --- |
| Domain commands and persistence | 02 defines schemas, guards and the unit of work. 03 implements hosted adapters. | Scoped principal, IDs, canonical JSON, integer money, business clock versus recorded time, command errors, idempotency, audit and outbox. |
| Covers, payouts and artifacts | 02 owns cover exports and claim orchestration. 05 owns engine semantics and T9. 03 provides durable registry and scoped storage. | v1 cover export and trusted scope and exact snapshot cutoff, explicit realm/register identity, binding and readback shapes, realm-bound artifact parents, and payout/cash-flow test vectors. |
| Voting and governed execution | 04 owns ballot privacy and tally. 02 owns proposal state, the claim decision slot and execution. 03 owns durable close and publication infrastructure. | Frozen draft/electorate, private ballot storage and outcome command. Confirm Plan 04's confidential-ballot recommendation with the fixture configuration and schema. |
| Rulebook and advice | 06 owns authoring, export and advice. 05 owns Rule IR, selection and sealed validation. 02 owns claim transitions and 03 owns the gateway. | Rulebook manifest, stable rule IDs, rule tests, read-only selector, advice snapshot and `SkipAiOpinion` authorization for the assigned assessor and authorized chair. |
| Settlement and replay | 07 owns journal recipes, reconciliation and dummy settlement. 02 owns money state and 05 produces cash flows. | Paid-due and final-payout causes, journal keys, `domain_flow` and `engine_cash_flows` fixtures, and zero opening cash for the first replay. |
| Claim intake | 08 owns web/channel adapters. 02 owns commands and access. 03 owns web/API infrastructure. | Start, edit, attach, submit and status payloads, a preallocated claim ID, first-channel provenance, exact retry across adapters and fixture-only evidence. Resolve the Plan 08 intake recommendations in the Plan 02 schema before adapter work merges. |

Plans 04, 06 and 07 contribute handlers to the same Plan 02 core. Assign each its records, handlers and tests, with Plan 02 owning the shared command catalogue, domain invariants and cross-aggregate processes. Plan 03 owns the migration sequence and adapter composition. Agree changes to shared schemas with their consumers before merging.

## 2. Build in parallel, then integrate

| Stream | Can start once | Work and handoff |
| --- | --- | --- |
| **02 — Domain core** | A0 domain contract | DM0 → DM1 → DM2 → DM3. Deliver the in-memory conformance runner and then tenancy, policy design, controlled simulation, member cover and dues. Hand Plan 05 the real v1 export with its pinned watermark at DM3. Deliver DM4 claims, decision execution, payouts and release state with fixture peers first. Verify the real engine handoff at A3. |
| **03 — Platform** | A0 persistence and scope contracts | Build A1 web/auth, PostgreSQL, audit and outbox alongside the in-memory domain. Run the same scenarios against both adapters as DM slices land. Start A2 registry, runner and job spikes alongside A1. Prove scoped publication, register CAS, lease fencing, cancellation and deterministic execution on the actual runner before relying on these adapters at A3. |
| **05 — Actuarial engine** | Shared artifact and fixture vectors | M0 → M1 → M2 locally, using fixture rulebook and cover adapters. Replace the M2 cover fixture with the exported DM3 snapshot for the domain/engine conformance run. Develop the C-7/C-8/C-10 T9 handoff against the M1 primitives and exercise it with 02/03 at A3. After M2, build M3 model depth while the platform integrates. Start M4 subscription work once the template interfaces stabilize, and complete its conformance against the M3 engine before A4. |
| **04 — Voting** | Proposal, clock and persistence contracts | Build P1–P3 electorate, ballots, close and deterministic tally using frozen fixtures. Wire claim voting to DM4 at A3 and the scoped close/tally worker to A2. Build P4's release coordinator against fixtures in parallel. Its end-to-end proof waits for G6 and the shared committee. |
| **06 — Rulebook and advice** | Rulebook export, Rule IR and claim snapshot contracts | Build P1–P2 authoring, immutable export and rule-test evidence alongside M1/M2. Build P3–P4 advice, skip and publication against fixture selectors and a test gateway. Integrate actual register readback, selector, AI gateway and claim/evidence authorization before A4. |
| **07 — Dummy ledger** | Ledger, cause and cash-flow contracts | Build P1 recipes and P2 dummy settlement using synthetic dues and committed payout fixtures. Integrate the domain-ledger path at A3. Build P3 replay independently against pinned Plan 05 cash-flow fixtures. Prove replay of the actual signed result and subscription renewal at A4. |
| **08 — Synthetic web claims** | Plan 02 claim commands and Plan 03 web conventions | Build P1–P2 forms, scoped evidence and fixture intake against the same command fixtures. Connect real DM4 handlers for A3. Build P3 receipt and status in parallel. Verify status against committed review, finalisation and settlement events before A4. |

Provisioning decisions have separate deadlines. Pick the identity provider and region before shared A1 deployment. Pin the runner image, CPU class and numerical versions for A2. Choose model families for the M2 extraction, independent coverage pass and generation work. Agree test-environment retention, backup targets and staff-data rules before A4. Local fixture work can start while these decisions are made.

## 3. Prove the first integrated claim at A3

Join the DM3 cover export, DM4 claim and proposal handlers, A1/A2 adapters, Plan 05 T9 handoff, Plan 07 dummy settlement and Plan 08 web intake. Use scoped fixtures for unfinished peers, as Plan 03 A3 permits. Run the T9 handoff against the actual engine adapter and authoritative register to close its acceptance cases.

1. Pay the fixture's first contribution and activate cover. Submit one synthetic web claim. Verify the committed receipt, original occurrence and discovery dates, report-time bundle and incident-date cover selection.
2. Register through the ordered realm queue using the v1 cover snapshot with its trusted realm and recorded cutoff. Keep the event ID fixed. Prove C-7 realm-bound artifact identity, C-8 unset flags and the committee-approved payout basis, and C-10 scoped readback of effective bindings, amounts and traces. Include pending rebind and cross-realm denial. Wait for registrations through the captured watermark before finalising.
3. Freeze and execute a committee decision. In the ledger fixture, approve 4,500 minor units and have T9 cap it at 4,000, independent of the preview. Starting with a 10,000 contribution, expected cash is 0 → 10,000 → 6,000. A rejection or zero final amount creates no payout.
4. Kill the worker after the T9 compare-and-swap, after settlement and after journal commit. Retry from the committed register or journal and prove one decision execution, one payout, one settlement posting and the original trace and balance. Run L1a/L2–L4, AT-CHAN-1/2 and the relevant DM4 conformance on the delivered adapters. Label fixture-only runs separately.

## 4. Complete the shared Phase 1 release at A4

**Resolve the C-11 milestone conflict before A4.** Plan 02 §23 and Plan 05 §26.3 list DM5 for full scheduled-segment support. Plan 06 P3 already requires the exercised report-date selector for shared G7. Pull the shared selector and its report-date filtering into A4, with a cross-plan acceptance case for a future segment whose Rule IR exists only in the next bundle. Keep governed schedule changes and the full DM5 cases in DM5. Agree this at A0 so Plan 06 can build against the contract immediately.

1. Use the scoped voting, rulebook/advice, web and ledger implementations on the shared path. Run DM4 second-type conformance after M4, and verify subscription renewal and the member status projection. Rehearse shared restore and erasure and record the results.
2. Complete M3 and obtain real G1–G5 sign-offs on the exact calculation bundle, including the full model-quality and sensitivity evidence. M2's `G5-rehearsal` only qualifies for its local fixture G6 check. Reserve engineering, analyst, owner and actuarial review time, plus specialist review where required.
3. In a separate `engine_cash_flows` realm, replay the signed primary result with zero opening cash. Verify AT-SIM-1 at every event, including zero-amount positions. Reconcile the `domain_flow` claim and payout journal separately. Bind the reconciled simulation report to the primary result and sign the matching release bundle at G6.
4. Run G7 on that release bundle. Copy the active live-realm committee users into the test realm, verify the frozen electorate and configuration still match, and record separate owner consent. Approval writes only the simulation configuration's `test_release`. Preserve the sign-offs, digest, voter snapshot, audit, run IDs and reconciliation evidence. Verify the live policy and real-money guards.

**Critical path to shared G7** goes through the A0 contracts, the slower of platform/DM4 and engine handoff readiness at A3, and the slower of A4 integration and genuine G1–G5 evidence. G6 then precedes G7. Starting the four feature streams early shortens the join only if they share the contract fixtures and can replace them with scoped adapters.

## 5. Sequence after the synthetic release

| Delivery | Gate |
| --- | --- |
| **Before Phase 2 — DM5 and M5** | Complete DM5 contract work for native C-1/C-2 v2 exports, dated proposals and application (C-5/C-6/C-9), and the full C-11 cases. Resolve U-DM-19 with the owner and committee before freezing step-limit checks. In parallel, build C-3 per-claim live bindings and C-4 audited event relinking, live web/enrolment and evidence controls (08), and custody, payment exceptions and reconciliation readiness (07) once their policy and provider decisions are agreed. Verify positive G8 in a separate live-capable environment. Actual launch requires G8 on the same tested release bundle with legal, partner and required future-work evidence. |
| **Phase 3 — full governance and recalculation** | Reuse the DM5 execution guards, native exports and dated proposal contracts. Complete Plan 04 P5 workflows and Plan 05 M6 recalculation, then enable each governed change with its signed proposal evidence and effective-date rules. |
| **Phase 4 — MCP and partner API** | Build Plan 08 P4 delegated adapters on the shared claim commands and verified live web workflow. Confirm client and user authorization, first-party declaration consent and per-client conformance before enabling them. |

## Source milestones

- [Plan 01 — Suggested phases](01-platform-overview.md#12-suggested-phases)
- [Plan 02 — Domain milestones](02-domain-model.md#19-milestones) and [engine change requests](02-domain-model.md#23-changes-requested-to-plan-05)
- [Plan 03 — Architecture milestones](03-architecture-and-tech-stack.md#9-milestones-and-acceptance)
- [Plan 04 — Voting and governance](04-voting-and-governance-engine.md)
- [Plan 05 — Engine milestones](05-actuarial-engine-and-simulation.md#27-milestones) and [Phase 1 handoff](05-actuarial-engine-and-simulation.md#263-phase-1-handoff-and-later-gates)
- [Plan 06 — Rulebook and advice](06-rulebook-and-ai-second-opinion.md)
- [Plan 07 — Payments and ledger](07-payments-fund-ledger-and-payouts.md)
- [Plan 08 — Claim channels](08-claim-channels-and-mcp.md)
