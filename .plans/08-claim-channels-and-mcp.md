# Plan 08 - Claim channels and MCP

## Context

Phase 1 delivers the synthetic web flow for the fixed-term event fixture in
[Plan 03](03-architecture-and-tech-stack.md). The first live web pilot
requires G8 and a separate deployment. Member MCP and partner API access follow in Phase 4.

[Plan 01](01-platform-overview.md) §§7.9, 7.11–7.12 and 7.15 set the scope.
[Plan 02](02-domain-model.md) §§6.1, 6.7, 9.6, 10–12, 14.2, 14.6 and 16 define
the domain contract. [Plan 06](06-rulebook-and-ai-second-opinion.md) owns committee advice.
Plan 08 owns the claimant interface, evidence ingress and enrolment page access.

## [P1] Route claim intake through shared commands

- Place the channel adapters outside the Plan 02 core. Use its typed schema package to map
  start, update, attach and submit to `StartClaim`, `UpdateClaimDraft`, `AttachEvidence`
  and `SubmitClaim`. Derive tenant, project and realm from canonical records and current
  authorization. Resolve the claimant from the authenticated member's party linkage or
  active business representation. Verify the cover belongs to that party and scope. A
  principal acting for multiple parties selects an authorised cover; the request cannot
  nominate an unchecked party or impersonate a member.
- Carry the same opaque claim target ID, operation key, authenticated user and canonical
  typed payload through retries in any channel. Reserve the Plan 02-format claim ID
  before the first `StartClaim` attempt so a lost response cannot produce a new target.
  Normalize adapter inputs before hashing. Record the channel of the first successful
  `StartClaim` as claim provenance and exclude adapter transport metadata from the replay
  comparison. Authorization precedes idempotency lookup. A changed payload under the same
  key and target returns a conflict. A different principal has its own key scope. Capture the
  draft version displayed for edit and submission. An exact committed retry returns the
  original result even if the draft version has since moved.
- Submit the raw occurrence and discovery dates as known, applicable insured item, incident
  type, member loss estimate in `Money`, description reference, and an optional eligible
  loss event. Present the versioned declaration and require explicit member acceptance.
  The domain records acceptance and fixes `reported_at` from the realm clock. Plan 05
  determines the incident date and applicable cover segment. The registration process
  picks the bundle in force at report time from the submission snapshot. Channel validation
  checks input shape and ownership. Cover, duplicate, waiting-period and limit findings
  follow the domain review path. A finding does not adjudicate the claim.
- Recommend making the provenance and preallocated target rules explicit in Plan 02
  §§6.7, 11.1 and 14.6 when the command schemas are agreed.

**AT-CHAN-1.** A lost start and submit response followed by a retry through another adapter
under the same member principal produces one claim, one submission event and the original
receipt. Changing the amount under the same key conflicts. Revoking the representative first
denies the replay, and another principal cannot get the stored receipt. Concurrent draft edits
against one version have one winner. Supply an occurrence crossing a cover change and submit
after a calculation-bundle change. Verify raw dates, the domain report time and registration's
pinned bundle, with the incident-era cover selected by the engine.

## [P2] Deliver synthetic web intake and scoped evidence

- In the signed-in simulation UI, an authorised project owner or analyst selects a reviewed
  fixture and initiates a run. The scoped simulation driver acts for a synthetic party using the
  same command payloads. Staff retain their project roles. The driver cannot act in the live
  realm. Run start, update, attach and submit against one dummy fixed-term claim. The shared
  deployment accepts fixture IDs for descriptions and evidence, resolving them from the
  approved catalogue. There is no free-text claim or arbitrary-file upload in that deployment.
- Plan 02 §§12.4 and 14.6 authorize the driver to start, update, attach and submit claims
  for synthetic parties in its own simulation realm. Enforce that scope at each command.
  Keep staff read permissions separate. Show fixture run progress to its authorised initiator
  and restrict claim and evidence reads to Plan 02 roles and access policy.
- Attach a claim-scoped, committed evidence reference. Resolve scope and approved artifact
  identity again when attaching it. Return only the evidence metadata allowed for the caller.
  An evidence download checks current access and key state and records the read before
  returning bytes through a scoped application endpoint. Keep free text and evidence bytes out
  of transport logs, events and receipt responses. Preserve Plan 02 erasure markers and
  retention. A stored-but-unattached object is invisible through claim APIs.

**AT-CHAN-2.** Run the web fixture from draft through submitted and into the Plan 03 A3
integration. Refuse a non-catalogue fixture, live realm, real party, cross-claim evidence
reference and stale representative. Refuse downloads after access removal or recusal, log each
successful evidence read, and return the erasure marker when an authorised reference is erased.

## [P3] Publish a member receipt and status view

- Return a receipt only after the domain commits `SubmitClaim` and its idempotency record.
  Include opaque claim ID, committed `reported_at` and initial state. The client reuses
  the key after timeout. Map unauthenticated access, absent or inaccessible resources,
  stale drafts and malformed input consistently across adapters. Keep a request reference
  independent of the domain retry key.
- Add a member-scoped status projection with current state, requests for additional
  evidence, and authorised decision reason, cited rule IDs and financial outcome when
  recorded. Show status changes after they commit. Exclude evidence contents, committee
  notes, ballots and AI advice. Keep the committee decision amount distinct from the final
  amount and the settled status. Surface the recorded explanation for zero final payout.
  Publish status notifications through the durable outbox. Send a generic notification
  and point the user to the authenticated status view for details. Simulation notifications
  go only to authorised real project-role users as specified in Plan 02 §15.4.

**AT-CHAN-3.** With registration deliberately delayed, a successful submit already has a
durable receipt while the status remains `submitted`. A later approved decision,
finalisation and payout each appear only after commit. The claimant sees the communicated
reason and pinned rule references while the committee notes and AI opinion remain inaccessible.
Repeat after access revocation and after a zero final payout. No test notification goes to
synthetic contact details.

## [P4] Gate live enrolment and delegated channels

- Resolve a public enrolment page only to a published, currently open policy. A private
  page requires an expiring, revocable opaque grant scoped to the policy, realm and
  intended verified applicant. A project-authorised operator issues the grant and sees
  issuance and revocation metadata. Store a token hash, avoid URLs in logs and access
  analytics, and redeem into the signed-in session before redirecting to a clean URL.
  Check grant validity on both `CreateParty` and `Enrol`, with the real applicant's
  principal and Plan 02 guards. Serialize enrolment and revocation against this grant so
  a later revocation blocks new dispatch. A failed or interrupted enrolment can resume
  with the same authorised operation while the grant remains valid. Revalidate
  `enrolment_open` within the domain guard; the grant cannot override it. Unauthenticated
  access to a public page exposes public policy information only.
- For Phase 4, give the MCP and partner API clients distinct registration and per-user
  delegated authorization to the claims service. Validate token issuer, recipient, validity
  and action scope. Identify both client and user and recheck current party and resource
  authorization at each call. Map both clients to the same draft, submission and status
  application interfaces. Require first-party, explicit review of the exact declaration
  version and draft version before delegated submission. Keep this acceptance proof
  bound to the member, claim and version and revalidate it at commit. Use web evidence
  upload until client-specific evidence tooling is approved.
- MCP tools expose only draft actions, reviewed submission and status. Committee decisions,
  AI opinions, payment and administrative actions remain outside the delegated interface.
  Treat document and model text as data. The authenticated application confirmation is the
  source for member intent. Partner onboarding and all transport, client capability and
  consent details require a Phase 4 compatibility review.

**AT-CHAN-4.** A private link with the wrong applicant, scope, expiry or revoked grant is
refused before any new enrolment command. Concurrent revocation or enrolment has one
observable order. Public enrolment still requires an authenticated applicant and an open
policy. A cross-tenant claim ID, wrong token recipient, stale consent, changed draft after
confirmation or attempted decision tool call cannot read or mutate a claim. An authorised
member reads the same committed receipt in web and through the later delegated adapter.

## Out of scope

- **Live evidence ingress.** Before G8, specify quarantine, type and size validation,
  malware scanning, encrypted publication, retention, cleanup of unattached uploads,
  access and erasure, and any external AI processing approval. Verify rejection and
  recovery paths before allowing real evidence entry.
- **Provider-specific MCP and partner onboarding.** Before Phase 4, verify supported
  transport, client registration and consent against the selected providers. Set
  abuse limits, share only approved fields and require per-client conformance runs.
- **Disputes and fraud investigations.** FW-3 and FW-8 own the later workflows.

## Notes

On 26 November 2026, Cursor `composer-2.5` approved P1–P4 in round 1. The bridge returned
`ALL_APPROVED` with exit code `0`. The review confirmed the shared claim commands, scoped
simulation driver, evidence access, enrolment guards and Phase 1 fixture boundary against
Plans 02 and 03. The acceptance scenarios remain planned work.
