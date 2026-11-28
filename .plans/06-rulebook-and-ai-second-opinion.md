# Plan 06 - Rulebook and AI second opinion on claims

## Context

Cursor `composer-2.5` approved P1–P4 in round 1 on 26 November 2026. The review confirmed
alignment with Plans 02, 03 and 05. The C-10/C-11 integration and `SkipAiOpinion` authorization
remain gated as specified in P3 and P4. No changes or disputes were recorded.

Draft for Phase 1. Deliver the rulebook and claims advice for the synthetic fixed-term event
policy through the G7 test release in [Plan 03](03-architecture-and-tech-stack.md). The
subscription fixture remains part of [Plan 02](02-domain-model.md) domain conformance. Use
the product rules and human decision requirements in [Plan 01](01-platform-overview.md)
§§7.5, 7.7 and 7.12, the claim and design records in Plan 02, and the [Plan 05](05-actuarial-engine-and-simulation.md)
engine and artifact contracts. The simulation uses approved synthetic policy and claim
fixtures. Real staff can take part under the Plan 03 identity rules.

## [P1] Freeze and export reviewable rulebooks

- In the policy design module, use Plan 02's draft and freeze commands. The owner authors
  long-form wording, coverage, exclusions, limits, evidence requirements and a plain-language
  member summary. Store each source as Plan 05 §8 original bytes plus human-reviewed
  extracted text and hash, with document type, version and provenance. Include source and
  summary references and their hashes in the immutable `content_ref` manifest before
  freezing. Render the member summary for the exact version and keep the underlying wording
  accessible to authorised readers. Freeze governs the content. Plan 05 gates separately
  establish approval and readiness.
- Publish Plan 05 §26 `get_rulebook_version(project_id, id)` through the scoped artifact
  registry. Export the manifest, sources, versions and text hashes for any frozen or retired
  version still pinned by member cover, with authorised access rechecked on child references.
  Registry validation rejects changed bytes, missing referenced documents and foreign scope.
  Preserve version history independently of the current pointer.
- Adopt Plan 05 §16 Rule IR and its typed data dictionary directly. In the draft, mint a
  stable opaque rule ID per logical rule. Carry it forward for revisions of the same rule.
  Mint new IDs for added rules and for distinct successors after splits and merges, recording the
  lineage in the rulebook's review record. Keep retired IDs reserved and use Plan 05 §17.2 for
  dated variants within an IR. Review links use `rulebook_version_id` and `rule_id`, plus the
  matched effective variant when an ID has multiple dated variants.
- Accept `submit_rule_ir` and `submit_issues` as candidate artifacts. The design review
  shows the cited text beside each condition and parameter and lets the owner confirm intent
  and close issues with reasons under Plan 05 §16. Use its deterministic citation verifier,
  clause coverage pass, condition and conflict checks. Legal specialists confirm the cases
  specified in Plan 05 §16.7. Publish and attach the approved IR via write-once
  `AttachRulebookArtifact` after freeze. Source changes or corrections to attached
  artifacts produce a new version and new gate evidence. Never overwrite a confirmed artifact.
- Acceptance RB-1. A retired version still exports and renders its original words and rule
  IDs. A changed source hash, bad offset, uncovered clause, missing specialist confirmation,
  or cross-project artifact fails the corresponding validation or approval step. Revisions
  preserve existing references and are tested against the same ID rules.

## [P2] Bind the rule tests and sign-offs to the proposed version

- Use the Plan 05 §20 sealed oracle suite as the Plan 01 rule test cases. An independent
  owner or analyst author derives expected coverage and payout in minor units from the
  actual wording. Seal the expectations and source citations independently of generated
  code and its development examples. Test coverage, exclusions, limits, evidence and
  judgement flags, and incident date boundaries. Keep sealed cases out of generator
  context as required by Plan 05.
- Write the first passing `rule_test_run_ref` once after freeze and IR attachment. Build it
  from the Plan 05 §24.2 validation report. Pin the rulebook version and `content_ref`, tested
  terms/rulebook pair and IR, oracle suite, validation report, calculation bundle and its
  test environment. The reference is permanent evidence of a passing run for that rulebook
  version. A later run with changed inputs belongs to the new calculation bundle and its
  own validation report. The original attachment remains readable.
- At Plan 02 `OpenProposal`, verify the `rule_test_run_ref` and its coherent bundle
  evidence as well as the existing INV-33 presence check. For a governed change, require the
  proposal's own signed calculation bundle, Plan 05 G1–G5 under §24.4, and its exact
  version map and validation report to prove the sealed rule tests passed for the proposed
  terms and rulebook pair. The same passing report can serve both checks if the bundle agrees.
  Plan 02 §13 executes approved votes and controls when changes take effect for cover segments.
  Keep Plan 02's dated pointer rules and Plan 03's same-bundle G7 test release.
- Acceptance RB-2. Block a rule change when tests are absent, failed, or tied to a different
  pair, suite or test environment than the proposed bundle requires. A new passing bundle
  can use its own fresh report without replacing the write-once record. An authorised change can move the dated
  pointer. An earlier cover continues to resolve its earlier rules. A test release
  leaves the live deployment disabled.

## [P3] Prepare advice from the incident-date rules and pinned evidence

- Start after Plan 02 §14.2 records engine registration and the claim is `under_review`.
  The claim process resolves the register's **effective** calculation bundle and member
  cover set, including rebinds through the Plan 02 C-10 readback. Use the original
  registered `ClaimInput`, with unset judgement and evidence flags for this stage.
  Before the AI call, expose Plan 05 §17.1's existing selector through the Plan 05
  adapter as a typed, side-effect-free operation. Return either the matched cover
  segment, incident date, terms version, rulebook version, IR and active rule variants,
  or a typed selection issue. Use §14.2/C-11's report-date limit on candidate segments.
  The same engine selector serves this operation and payout previews. The readiness
  preview and later decision-draft previews keep their Plan 02 triggers.
- Snapshot the versioned claim input references, evidence IDs and hashes, and the
  selection result for one advice attempt. The gateway reads only approved synthetic
  evidence objects or vetted fixture text derivatives through Plan 02 §12.3. Treat
  source and evidence text as data, use a fixed prompt template and allowlisted schema,
  and provide the model with no tool or external resource access. Recheck the scoped
  inputs on fetch and before publication. The recommendation may relate only to the
  captured evidence and supplied active rules. Include evidence IDs for material
  statements, and explicitly call out missing information or possible inconsistencies.
- Validate the returned recommendation, confidence, evidence IDs and exact cited rule
  variants against the snapshot. Use `low | medium | high` as a proposed Phase 1
  confidence schema, labelled as the model's self-assessment. Leave the optional
  `suggested_amount` unset in Phase 1. Reject invented or inactive citations and
  supplied decision flags. Keep evidence checks and judgement flags with the humans
  authorised by Plan 02. An unmatched or ambiguous segment yields a recorded flag or
  issue and a reason to skip advice. A missing mapping or failed engine lookup remains
  a technical issue and cannot be recast as a coverage exclusion.
- Acceptance RB-3. Two covers on the same date can receive different valid cited rules.
  Exercise disjoint rule variants, discovery dates, a missing or ambiguous match,
  future segments, and a Phase 1 register rebind. A malicious evidence instruction
  cannot cause an action or inject a citation. Unknown, inactive and foreign evidence
  or rule IDs fail validation. Treat the C-10/C-11 adapter as a fixture contract until
  Plan 05 agrees to it. Shared G7 requires the corresponding exercised integration.

## [P4] Publish advisory records without creating a claim decision

- Use Plan 03 §7.2's AI gateway and durable job runner. Key one attempt by scoped
  claim input and evidence snapshot identity, prompt/model configuration and stage.
  Use the Plan 02 command idempotency key and Plan 03 lease fence for publication.
  Record each provider call with model and model version, prompt version, input artifact
  references, output and time. Where a call fails without an output, log the failure
  code and time. Publish the `llm_call_log` before `AttachAiOpinion`, binding its
  immutable identity to the claim realm using a verified realm-bound member cover set
  parent and any referenced realm-bound evidence parents (Plan 02 §5.7).
- Store the `AiOpinion` summary as personal text in the same unit of work as the
  authorised attachment. Encrypt AI logs and derivatives within the Plan 02 §16 and
  Plan 03 §6 claim erasure/retention scope. Keep raw prompts, evidence and free text
  out of audit, event and operational logs. Audit evidence and opinion access, including
  the relevant principal and record IDs, using Plan 02's claim visibility and evidence
  rules. A recused seat cannot open an evidence-derived opinion or its log.
- A late, superseded or cancelled job cannot attach to a changed claim snapshot or to a
  claim outside `under_review`. Show an attached opinion's evidence snapshot and mark
  it outdated when evidence or its applicable rule context changes. The assessor or
  chair can request a fresh attempt during review or record `SkipAiOpinion` with a
  reason when advice is unavailable or no longer current. Plan 02 should explicitly
  grant that skip command to the assigned assessor and chair. Use bounded job retries
  and surface the skip action after failure instead of delaying committee review
  indefinitely. `MarkReadyForCommittee` remains Plan 02 §9.6's separate command after
  checks and completion or skip are recorded. The gateway can attach advice only.
- In the committee view, display the evidence and pinned wording beside the labelled
  optional advice or skip status. Never copy its recommendation into the decision
  draft by default. The assessor or chair prepares a draft under Plan 02. Plan 04
  tallies the vote and Plan 02 writes the decision only on execution of an approved
  proposal. The final amount remains Plan 05's post-decision calculation.
- Acceptance RB-4. Duplicate responses create one attached opinion. Late responses
  cannot modify a newer snapshot or a decided claim. A provider failure reaches
  committee review through a recorded skip. Every recommendation leaves decision, state
  and amount unchanged at attachment. Run this alongside AT-DM-CLM-3/4/7/8 and a G7
  simulation trace. Assert cross-realm, recusal, live-evidence and erased-key denial.

## Open questions

| Deferred scope | Owner and gate |
| --- | --- |
| External AI on real member evidence | Privacy/legal and platform engineering resolve Plan 02 U-DM-15 and Plan 05 U-13. Require a jurisdiction-specific decision covering allowed data, processing location, provider retention, audit and erasure before any external live evidence processing. Phase 1 denies it at the gateway. |
| Conflicting or ambiguous terms in live claims | Product and legal define human escalation, reasons and review timelines with FW-3 and FW-6 before G8. Phase 1 can flag or skip advice and preserve the existing committee and engine contract. |
| Automated evidence extraction beyond the vetted synthetic fixtures, fraud investigation and rule change impact replay | Maintain separate work items under Plan 01 §§8 and 10. FW-8 owns investigation and claimant response. Reuse version-pinned sources and independently verified results when they are specified. |
