# Plan 04 - Voting and governance engine

## Context

Deliver Phase 1 voting for synthetic claims and the G7 test release. Use
[Plan 01](01-platform-overview.md) §§7.6, 7.12 and 12, [Plan 02](02-domain-model.md) §§6.4,
8.2, 9.4, 9.6, 12–15 and 20, [Plan 03](03-architecture-and-tech-stack.md) §§3–5 and 8,
and [Plan 05](05-actuarial-engine-and-simulation.md) §§14.3 and 24. Plan 02 owns commands,
aggregate transitions and proposal execution. Plan 04 owns ballot mechanics and tallying.

## [P1] Freeze the question and electorate

In Plan 02's Proposal command handlers, freeze the typed question, baseline and governance
settings version at creation. Show the exact decision draft before a committee member casts
a ballot. For claims, a `yes` approves the whole draft, including its disposition, amount,
reasons, cited rules and judgement flags. A claim denial requires a draft with disposition
`reject` and an approved vote. A failed or rejected vote returns the claim to
`ready_for_committee` through the decision-slot release process. Comments and AI advice
can inform a voter. Neither can change the draft or cast a ballot.

`StartVoting` takes one commit-ordered realm snapshot, checks Plan 02's committee limit
and refresh guards, and freezes the sorted, distinct `voter_ref` set of eligible active seats
after recusals. Capture the committee snapshot watermark, settings ID, `opened_at` and
`closes_at` in the proposal. Evaluate the chair's eligibility by the same rules as other seats.
Keep the electorate fixed through the vote. Recheck scoped principal authorization at cast,
including Plan 02 role conflicts. A later seat addition cannot join the ballot. An already
accepted ballot remains counted if the seat ends. Use the existing claim-evidence access
checks on each read.

Create automatic recusals for the claimant's linked user and representatives per Plan 02.
Accept declared recusals in draft or discussion before `StartVoting`, within the same
proposal concurrency boundary. Check the current claimant links at opening so a newly
recorded representative is excluded before the freeze. If this leaves too few eligible
voters, preserve the claim and explain `min_eligible_voters` in the status. Phase 1 fixtures
keep claimant links fixed while voting is open. Live handling of a newly discovered conflict
after the freeze is a FW-6 launch gate for the affected flow.

Verify Plan 02 AT-DM-GOV-1–4 with overlapping seat changes, automatic and declared recusals,
an unrefreshed committee, and two attempts to open a claim decision. Assert that a stale claim
draft cannot execute after new evidence and that a negative vote on a draft creates no claim
decision or payout.

## [P2] Collect and protect ballots

Use `CastVote` on the Proposal aggregate, scoped to `(tenant_id, project_id, realm_id)`.
Accept `yes`, `no` or `abstain` during `[opened_at, closes_at)` by authoritative realm time.
Use Plan 02 S-1 idempotency and version checks. Enforce one accepted current ballot per
`voter_ref`. Named ballots default to one submission. A frozen `allow_vote_changes` flag can
allow an eligible voter to replace their named ballot before close. Concurrent replacements
with the same expected version have one winner. Both an original submission and a replacement
have a stable command key for exact retries. The tenant isolation and role rules apply to every
write and read, including the retry path. The simulation driver may act only for scoped synthetic
voters. Real users cast their own ballots.

For U-DM-14, recommend confidential ballots for Phase 1. An atomic anonymous vote writes a
voter participation marker and a separately generated, randomized encrypted choice in an
unordered set. Bind ballot encryption to the tenant, project, realm and proposal. Give the
casting service encryption capability only. The marker carries no ballot identifier. An anonymous
choice has no voter, comment, timestamp or vote-replacement history. Disable anonymous replacements and
ballot comments in Phase 1. Register the private records under the same aggregate commit
boundary and retain only participation metadata in identity-bearing audit and idempotency
projections. Do not put a bare hash of a three-choice value in an event, audit entry, command
log or idempotency record. Bind retries to the request with a server-keyed digest inaccessible
to report readers. On close, authorize the tally service to read only the final anonymous
ciphertext set and the marker count, then decrypt choices through Plan 03's managed-key
boundary. Validate each choice and its scope. Enforce unique participation markers with a
database constraint. If the marker count and decrypted ballot count differ, stop and record
an operational integrity error before submitting any outcome.

Authorized named-vote views may show voter and choice. For anonymous proposals, report totals
after close and aggregate participation while open. Restrict participation markers, ciphertext,
command logs and historical revisions to service access. Auditors verify the application audit
chain, settings, participation proof and final totals without a voter-choice view. Suppress
per-voter participation in ordinary reports. Published totals can reveal preferences when a
small committee votes unanimously. Product wording must disclose that limit. Phase 1 does not
claim anonymity against privileged operators combining storage history, service secrets and
live request observation. Hostile-operator secrecy and anonymous replacements are placeholders
for a dedicated ballot privacy design before any deployment that promises them.

Verify one named replace, identical retries, competing submissions, cross-proposal ciphertext
rejection, plaintext and bare-hash absence from ordinary logs and projections, and project-auditor
denial of the private data.
Run the inspection against historical and current read paths as well as the primary endpoint.

## [P3] Close voting deterministically and record the outcome

Add a deterministic tally to the Plan 02 core and a Plan 03 realm-scoped worker for
`StartVoting` timers and `RecordProposalOutcome`. On start use a positive frozen voting
duration. Public and anonymous ballots use the same tally. When the interval closes, use
the final current ballots and frozen rules from one commit-ordered Proposal snapshot.
The sandbox clock schedules the close at `closes_at` under Plan 02 §15.2. For later live
deployment, a durable due-time trigger rechecks the authoritative clock and accepts only
ballots recorded before the deadline. Never close early based on interim counts.

| Rule | Phase 1 behavior |
|---|---|
| Electorate | The frozen eligible-voter count is the quorum denominator. |
| Quorum | `yes + no + abstain`. Support an absolute minimum of at least two, or a percentage in basis points rounded up with a floor of two. Reject invalid ranges at settings validation. If fewer ballots arrive, record `failed` with `no_quorum`. |
| Abstention | Counts toward quorum. Exclude it from the approval denominator. If nobody casts `yes` or `no`, record `failed` with `no_decisive_ballots`. |
| Majority | Default strict majority of decisive ballots (`yes > no`). Support a supermajority from 5,001 to 10,000 basis points via `yes * 10000 >= (yes + no) * required_bps`. Use integer arithmetic. |
| Tie | Default `rejected`; allow a settings choice of `failed` with `tie`. There is no extra chair ballot in Phase 1. |
| Independent approvers | A preliminary approval requires at least two distinct eligible `yes` voters. Otherwise record `failed` with `too_few_approvers`. Plan 02 independently enforces the count and rejects configurations that permit a one-person approval. |
| Release consent | Owner consent is separate from ballots and must be recorded before `closes_at`. A would-be approval without it becomes `failed` with `owner_consent_missing`. |

Write a realm-bound immutable tally reference with the proposal ID, frozen settings ID,
snapshot version and watermark, deadline, electorate and ballot counts, arithmetic and tie
rule version, and outcome reason. Exclude choices linked to individual voters from the tally.
Use content-addressed immutable publication outside the command transaction. Submit
`RecordProposalOutcome` with the three counts, scope, tally reference and expected Proposal
version. Validate artifact scope and snapshot binding. Compare named counts to the ballots.
For anonymous ballots, accept choice counts from the scoped tally service and check that
their sum matches the accepted marker count. The command refuses an early or stale close.
Use a stable key for this proposal/deadline and
retry from the stored committed outcome or re-read and recompute after a version conflict.
A different snapshot can produce a different immutable tally reference before a commit,
while a lost success response recovers the committed outcome without changing it. Keep the
tally, outcome, audit and outbox pointer atomic under Plan 02 S-1 and Plan 03 S-1/S-7.
The voting worker cannot execute a target write. Consume `ProposalApproved` through the
existing process runner, with the domain's own target idempotency and stale/expired rules.

Use table-driven tests for zero turnout, abstention-only, percentage rounding at quorum,
one `yes` plus abstention, a tie, and exactly two anonymous `yes` ballots. Race final
cast, timeout, simulation clock advance and retry. Inject a crash after tally publication and after
the record commit. Require one outcome, one approval event and at most one target execution.
Keep AT-DM-GOV-14 as an independent domain-layer check even though valid settings cannot
produce a one-person approval.

## [P4] Exercise the G7 test release with live committee users

Create a dedicated simulation realm and copy exactly the active live-realm committee users
and seat roles from a recorded snapshot. Use separately scoped server reads to compare the
source committee and test electorate. Keep the source snapshot and proof accessible only to
the authorized G7 coordinator. Start only when the copied set still matches and the frozen
electorate will contain exactly those users. If the copied set no longer matches, restart the
G7 setup. Reject synthetic G7 voters and use the normal ballot path for the copied users.

Bind the proposal to the same release bundle, calculation bundle and four in-force design
version IDs required by Plan 02 §15.4. Require the valid G6 evidence already specified by
Plans 02, 03 and 05. Open the release into `discussion` even with zero discussion time. The
domain process takes `LockForRelease` before `StartVoting`. Record owner consent before close.
On approval, Plan 02's process runner executes the test release in the simulation configuration
and clears the lock in that one target write. The write records `test_release` with its bundle
and proposal ID. Sign-off evidence carries the source committee snapshot, eligible set proof,
release bundle and test proposal ID. Route later live G8 through a separately deployed flow
with its own same-bundle legal readiness checks. Phase 1 has no live transition or real-money
capability.

Run Plan 02 AT-DM-REL-1 and integrated Plan 03 A4 with copied-seat mismatch, synthetic voter,
missing/late owner consent, mismatched release bundle, missing lock, and a lost execution
response. Verify that only the simulation test-release target is executed and the live
configuration, project lifecycle and real-payment boundary remain intact.

## [P5] Map governed changes onto the same ballot path

Exercise the generic Proposal and tally with synthetic fixtures before enabling each live
decision type in its planned phase. The calculus, target baselines and timing remain the
existing Plan 02 §§13.1–13.4 and Plan 05 §§14.3 and 24.4 contracts.

| Type | Execution handoff |
|---|---|
| `contribution_change` | The actuarial engine can propose a signed calculation bundle and a dated schedule. The committee votes and the configuration domain enforces notice, baselines and step limits. |
| `rule_change`, `terms_change` | The domain checks pinned versions and applicable calculation evidence. Rule changes require the Plan 06 rule test result before voting. |
| `governance_change` | Use the settings frozen for this proposal. An approved execution can only schedule the replacement settings for future proposals and refresh the committee limits through the existing process. |
| `committee_change` | Target the Committee and enforce the current seat limits. |
| `member_removal` | Target the membership with the required reason, notice and scheduled effective date. |

Run Plan 02 AT-DM-GOV-5–13 as applicable and Plan 05 AT-GOV-1, AT-GOV-4 and AT-GOV-5.
Cover concurrent approvals, duplicate execution, a governance change during another vote,
and an approved date that expires before execution. Reuse the typed command and outbox
interfaces. The dashboard and notifications read scoped projections and outcome events.

## Out of scope

- **FW-2 member democracy.** Design member electorates, delegation and protected rights before
  offering member-wide decisions. Phase 1 and the generic extension above use committee seats.
- **FW-6 committee design.** Resolve live composition, disclosed and late conflicts, recusals
  that leave too few voters, deadlock escalation, chair powers, and committee compensation
  before live deployment of affected workflows. No chair override or alternate decision maker
  in Phase 1. Any future tie-break must preserve Plan 02's two actual eligible `yes` votes.
- Appeals and lifecycle decisions remain with FW-3 and FW-5. Legal readiness and the
  authorization for live claims handling are part of G8, outside this voting design.

## Notes

On 26 November 2026, the read-only Cursor `composer-2.5` review approved P1-P5 in one
completed round. Cursor confirmed alignment with Plan 02's proposal, claim and committee
contracts, Plan 03 S-1, S-7 and A4, and Plan 05 AT-GOV-1, AT-GOV-4 and AT-GOV-5. No changes
or disputes were recorded.
