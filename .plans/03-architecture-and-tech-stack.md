# Plan 03 — Architecture and tech stack

| Field | Value |
|---|---|
| Status | Draft v0.2 |
| Date | 2026-11-25 |
| Scope | Phase 0 foundation and Phase 1 design, simulation, and G7 test release |
| Inputs | [01 Platform overview](01-platform-overview.md), [02 Domain model](02-domain-model.md), [05 Actuarial engine and simulation](05-actuarial-engine-and-simulation.md) |

## 1. First delivery

An owner signs in, creates a tenant and project, and uses a synthetic fixed-term event policy to
run the design and simulation flow. An analyst reviews the actuarial results. A simulated member
submits one claim, the committee approves an amount, the actuarial engine finalises it against the
pinned terms and applicable limits, and the dummy ledger records the payout. The owner and
committee then exercise G7 with the signed release bundle. Start with one event fixture. Plan 02
still requires both fixture types for domain conformance, and the live pilot product remains open.

The shared Phase 1 deployment imports policy, dataset, member and evidence data from the
reviewed fixture catalogue. Its import and evidence endpoints resolve approved fixture IDs.
Staff sign-in and invitations may use real identity metadata under section 6. Use a separate
deployment, identity client, database, buckets, keys, and runner infrastructure for any future
Phase 2 live operation. Phase 1 builds omit the live payment provider adapter and credentials.
The command composition rejects live enrolment, collection, payout, and G8 execution. The
project initialisation creates the live realm and its Committee. Before the G7 test, the
owner fills that committee's active seats with real project users.

## 2. Stack and deployment

| Component | Decision |
|---|---|
| Browser and API | Python 3.12, FastAPI, server-rendered HTML and forms for the first workflow. Versioned JSON command and query endpoints for the browser and test driver. Reuse Plan 02 schemas and command errors. |
| Domain core | Use the framework-independent Python reference core from Plan 02 directly. Put access and persistence adapters outside it. Run the same conformance scenarios against in-memory and database adapters. |
| Database | PostgreSQL 17 with explicit transactions and versioned SQL migrations. Authoritative structured state, personal ciphertext, audit, outbox, job leases, checkpoints, artifact metadata, and engine register heads. |
| Object bytes | Amazon S3 general-purpose bucket in the shared environment, through a small object store port. Use Plan 05's filesystem registry for local fixtures. Run hosted-adapter checks against the real object store. |
| Identity | Hosted OpenID Connect provider with MFA. OIDC subject mapping and application-managed tenant and project memberships. Select the provider before shared deployment. |
| Runtime | One repository and application release, with separate web/API and background worker processes. A trusted engine worker imports Plan 05 and uses a dedicated Linux amd64 runner pool for generated code. Use gVisor as the candidate runner runtime, gated on section 7. |
| Deployment | One region for the shared environment, with private database and runner connectivity. Deploy API and worker as separate container processes with distinct identities. Use a Linux runner VM pool. Local Windows development can use a remote Linux runner for isolation and reproducibility checks. |

The web process authenticates requests, calls domain handlers, and enqueues work. It serves
responses from persisted state. Workers advance durable workflows, call the AI gateway or
engine, and report results through Plan 02 commands. The API, domain worker, AI gateway, engine
worker, runner launcher and migration role each get only the privileges required for their tasks.
API and worker rollouts use backward-compatible schema changes and drain or fence old workers
before removing old fields. Infrastructure sizing follows measured fixture runtime and job
concurrency. Account for object storage, requests, backup retention, transfer, and per-key costs
when sizing the shared deployment.

## 3. Authorization and isolation

- Use the OIDC authorization code flow with PKCE. Validate state, nonce, issuer, audience,
  signatures and expiry, map users by issuer and subject, and keep session identifiers in secure,
  HTTP-only cookies. Protect browser mutations against CSRF. Read current application roles for
  each command. The identity provider establishes identity, and Plan 02 authorizes actions.
- Resolve `tenant_id`, `project_id` and `realm_id` from validated membership and canonical
  records. Check the action, resource and realm in the handler. Job requests carry the authorized
  scope, initiating user or cause, and operation. A worker authenticates as its own service
  principal, validates the stored scope and cause, and invokes only its permitted commands.
  Reauthorize at user-delegated steps and at status, cancel, resume and artifact access.
- Use composite keys and foreign keys for scope consistency. Apply PostgreSQL row-level security
  to tenant rows, and project and realm context on tables at those scopes. Use transaction-local
  context populated by trusted application code after the scoped authorization checks. Missing
  tenant or project/realm context denies access to rows at that scope. Keep global identity and
  tenant discovery behind a narrow lookup keyed by the authenticated subject. Test pooled
  connections for context reset, and run normal traffic under non-owner roles without
  `BYPASSRLS`. Enforce RLS for table owners as well.
- Give each worker only the scope attached to its claimed job. For lease discovery and outbox
  dispatch, expose bounded internal database operations that return only routing metadata or
  lease IDs for that worker type. Resolve all bodies in a scoped transaction. Access to support
  data through break-glass requires an individually authenticated, time-limited operator grant
  for a named scope, with reason and audit. Keep migration credentials out of runtime processes.
- Trust realm kind and money kind only from persisted domain state. Dummy-realm payout and
  payment provider calls fail at the adapter boundary. Synthetic settlement uses Plan 07's
  dummy posting flow inside the domain. Plan 02 owns allowed shared design references.

## 4. Persistence contract

| Plan 02 property | Implementation |
|---|---|
| S-1 Atomic updates | One PostgreSQL transaction checks the idempotency record, performs the conditional aggregate write, saves its new immutable version and personal ciphertext changes, appends audit and outbox records, and stores the command result. Return the stored result for an exact replay in the same Plan 02 idempotency scope. Refuse a different payload under the same key. Return `conflict` for a stale expected version. |
| S-2 Audit stream | Allocate a transactional gapless ordinal by locking the stream head and appending the hash-chained event. Store the head and event in the same commit. Runtime roles may append through the write path and read authorized audit records. Prohibit direct updates and deletes of audit rows. |
| S-3 Immutable objects | Conditional create and verified immutable publication via section 5. |
| S-4 Historical snapshots | Serialize realm writes on a watermark row. With the lock held, assign `recorded_at` as the greater of current UTC time and the previous value plus one microsecond. Update the watermark in the same transaction as the new aggregate version. All writers for a realm take that lock before checking commit-time guards and updating aggregate or stream heads. Use a single lock order for all commands. |
| S-5 Scoped storage | Apply the policies and constraints in section 3. Test with the deployed service roles. |
| S-6 Uniqueness | SQL constraints for every key in Plan 02 §18, plus Plan 02's journal idempotency key per ledger. Declare live-realm uniqueness per project, and keep one immutable realm binding per artifact ID. |
| S-7 Event delivery | Transactional outbox, ordered by aggregate version and event ordinal. Claim only each consumer's next unacknowledged event for an aggregate. Advance its cursor after successful handling. A lease loss can repeat a delivery. Process handlers record a stable operation key derived from event and step, and retry with that same key. |
| S-8 Erasable data keys | Envelope encryption and a key service with non-exportable keys per erasure group, as in section 6. |

Plan 02's project and realm initialisation steps use committed events and stable keys. The
original write stays visibly `pending` until every expected record exists and the ready command
commits. Idempotency makes a worker crash between the steps recoverable. Long-running jobs and
object I/O run outside command transactions.

Read a historical snapshot in one read-only repeatable-read transaction. Read the committed
realm watermark first, reject a cutoff greater than that watermark, and resolve each aggregate
from its latest version at or before the cutoff. Capture the watermark for default exports.
Preserve recorded time separately from the simulation business clock. Let realm commands that
use business time read it while holding the realm lock. Bind live command business timestamps
to the assigned `recorded_at` under Plan 02 §5.5. Join only frozen project-shared design versions
by ID. Historical aggregate snapshots contain personal reference IDs and salted hashes as
specified in Plan 02 §16, with no personal value joins. Historical exports use those references
and show the current `erased` marker with `erased_at` where applicable. Resolve personal
values for authorized current views separately against current permissions and key state.
Append an audit event before returning claim evidence.
[PostgreSQL transaction isolation](https://www.postgresql.org/docs/17/transaction-iso.html) ·
[PostgreSQL row security](https://www.postgresql.org/docs/17/ddl-rowsecurity.html)

## 5. Artifact storage and engine registers

1. The trusted registry checks artifact type, canonical Plan 05 ID, authorized scope, content
   digest and parent references. Put bytes under a scope-specific key with conditional create.
   On an existing key, verify the stored version against the original scope and canonical
   content identity. Reuse the original ciphertext and checksum on an exact retry, since
   encrypting the same content again can produce different bytes. Pin the S3 version ID and
   stored-byte checksum. Private buckets enforce conditional writes on final keys and deny
   deletes to runtime identities. Encrypt objects and transport. Scope role and bucket
   permissions to the registry operation and environment.
2. Record availability in PostgreSQL only after verifying durable bytes. Artifact consumers
   always resolve through committed metadata and read the pinned immutable version. For
   realm-owned artifacts, validate the Plan 02 §5.7 realm proof and publish the metadata and
   unique realm binding together. Conflicting or missing proofs fail closed. Verify the
   tenant/project/realm again in the engine before input staging. Platform template releases
   and other allowed project-shared artifacts follow Plan 02's separate scoping rules. Check
   authorization on each resolved reference, including children of a shared bundle.
3. If a worker dies between the object write and the metadata commit, the object remains
   invisible to readers. Retry publication using the same object and verified version. Track
   stale unpublished writes for reconciliation. Publication and retention acquire the same
   registry reservation for that key. A retention worker first marks the row unavailable and
   fences new publication, then deletes only the pinned object version after checking references,
   legal holds and backup policy. Preserve a tombstone so a retired identity cannot be published
   again.
4. Store the immutable T9 register versions and trace references. One compare-and-swap head per
   `(tenant_id, project_id, realm_id, register_id)` permits distinct simulations per project.
   The trusted T9 writer publishes the version and trace artifacts first, then advances the
   head, operation receipt and final result/trace references in one database transaction.
   Lock scope and verify the expected head. On conflict, re-read and recompute against the
   latest head. A lost response is recovered from the receipt and register readback with the
   stored effective binding. The domain then records the final amount idempotently.

Plan 05's register interface describes a conditional pointer per project and separate registers
per context. The scoped adapter supplies conditional update per context and requires Plan 05 to
accept the explicit realm/register identity. Plan 02 §23 C-7, C-8 and C-10 are dependencies for
the integrated claim. Until they are agreed, exercise storage and the contract with fixtures.
[Amazon S3 conditional writes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html) ·
[Amazon S3 security practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)

## 6. Personal data and recovery

The personal store has one row per logical record, separate from versioned aggregates. Each
row holds current ciphertext or an `erased_at` marker. Updates overwrite the current row
in place. Persist changes in the same transaction as their domain references. Keep plaintext,
unsalted free-text hashes, credentials, and evidence out of aggregate history, audit, outbox,
and operational logs. For Plan 02 §16
erasable data, keep only wrapped data keys plus the key service reference in PostgreSQL.
Encrypt object evidence and any authorized derivatives before uploading. Use authenticated
encryption with scope and object identity as associated data. Plaintext keys and worker scratch
exist only for the active operation. Avoid persistent plaintext caches, downloads and backups.

For the small shared deployment, use a single-region, non-exportable AWS KMS wrapping key per
erasure group. PostgreSQL records the stable group identity and the wrapped data keys.
Use staged key provisioning with a unique group reservation, reconcile failed provisioning before
use, and keep the decrypt and erase roles separate. Group objects only under Plan 02's subject
and retention rules. Begin erasure by blocking reads, draining in-flight access, and marking the
group pending. After the retention and hold checks, disable and schedule deletion of its key.
Record `ErasureCompleted` after irreversible deletion is confirmed. An erasure request stays
pending while deletion can still be reversed. Keep a durable, separately recovered erasure
journal of blocked and completed group IDs. A restore must apply the latest journal and check
the current key state before enabling reads, and fail closed if either is unavailable. Confirm
an old backup cannot decrypt an erased group. Key quotas and cost must be reassessed before
deployment with real policy data.
[AWS KMS key deletion](https://docs.aws.amazon.com/kms/latest/developerguide/deleting-keys.html)

Back up PostgreSQL, including history and outbox cursors. Pin referenced S3 versions through the
database recovery window. Before shared release, rehearse an isolated restore. Verify the audit
chain and watermarks, check referenced artifact digests, apply erasures, validate queue cursors
and leases, and replay a known synthetic workflow. Use a separate recovery identity and record
the measured recovery time and data loss. Keep operational alerts for audit verification
failures, job backlog and lease churn, sandbox refusals, backup failure and failed erasure.

## 7. Engine execution and jobs

### 7.1 Runner

The trusted launcher accepts jobs over a private authenticated channel. It verifies the
requested scope, stage, exact input manifest, runtime profile, image digest and file checksums.
Stage only authorized inputs, the template and pinned numerical environment, with fresh
per-run read-only mounts. For oracle runs, inputs contain claims and required reference data.
Keep sealed expectations, validation outcomes, and signing material in the trusted validator.

Use a dedicated Linux host pool and one fresh gVisor sandbox per run with external networking
disabled and sandbox network socket use denied. Enforce network denial at the host as well.
Give the guest an unprivileged UID, a restricted syscall profile, no extra capabilities, a
read-only root, and a single empty quota-backed output directory. Pass no tokens, host services
or devices. Set cgroup limits, per-process requirements, disk quota and an external wall-clock
watchdog. Validate bounded regular output files against the schema after terminating the sandbox.
Reject unsafe paths, links, device files, overlarge output and invalid schemas. The trusted
launcher reports bounded execution metadata. Destroy transient inputs and outputs before the next
job. Only the trusted validator can publish a result.

Pin amd64 CPU class, the specific Python patch and numerical wheel hashes, image digest,
BLAS build and CPU dispatch profile, seed and runtime settings in the first `environment_lock`.
Run the Plan 05 determinism probe before use and compare the full fixture outputs across two
workers. Pin the exact compatible versions in the A2 spike. Adoption requires passing Plan 05
ISO-1–ISO-9 and associated acceptance tests on the actual runner. Keep generated workloads
disabled if any isolation or reproducibility gate fails.
[gVisor networking](https://gvisor.dev/docs/user_guide/networking/) ·
[gVisor security model](https://gvisor.dev/docs/architecture_guide/security/)

### 7.2 Jobs and AI gateway

- Persist job state, stage, scope, operation, progress log, checkpoint ID, ETA when measurable,
  and cancel request. Provide authenticated submit, status, cancel and resume operations. Workers
  claim with expiring leases and fencing tokens. Late workers cannot commit checkpoints or
  results. Retain bounded retries with backoff for transient failures. Record explicit failed,
  cancelled, waiting-for-human and done states.
- Publish immutable checkpoint bytes first. Update the visible checkpoint pointer, progress
  log and produced-work receipt atomically after comparing the lease, state and input hash.
  Cancellation wins if its state commits before publication, and the worker stops its sandbox
  or discards a late result. Resuming validates the input hash and starts from the last committed
  checkpoint. Resource-limit failures require an explicit reviewed change before rerunning.
- Preserve Plan 05 §25 reuse keys and distinguish a retry from a new LLM iteration. Verify the
  artifact's allowed scope on reuse and bind realm-owned results to the realm through their
  identity-defining inputs. Accept a Monte Carlo chunk once by ID. Ignore identical duplicates
  and fail the job on divergent bytes. Retry after an ambiguous response with the same
  durable operation identity.
- The AI gateway alone may reach the configured model provider. It accepts schema-defined
  synthetic inputs, versions the prompt, records Plan 05 call logs under artifact access rules,
  and returns draft outputs. Calls get bounded timeouts, rate limits and per-job attempt caps.
  The gateway has no vote, approval, release, payout or database administration privileges.

## 8. Workflow and dependent plans

1. Materialise a design and simulation realm. Freeze inputs, run the model, and sign the
   calculation bundle through G1–G5 under Plan 05.
2. Use Plan 06 rulebook versions, stable rule IDs, and claim intake and AI advice contracts.
   Pin the calculation bundle and cover export under Plan 02 §14.2. Feed registration through
   an ordered queue per realm using the domain's report-time order. Before finalising, wait
   for required registrations at the captured watermark and reconcile previously committed
   final amounts using T9 readback. For the first fixture, keep the event ID fixed at registration.
3. Plan 04 produces the committee decision from the frozen voter set and reports a result through
   its authorized command. Apply Plan 02's minimum voter rule for claim and release decisions.
   Send the approved amount and decision reference to T9. Finalise per Plan 05 and record the
   authoritative trace and amount before creating a payout.
4. Execute the Plan 07 dummy settlement in the `domain_flow` realm and post the journal entry
   once per settled payout. Inject a lost worker response after settlement and prove the retry
   cannot create a second payout or journal entry. An engine-only simulation uses a different
   realm and posts only `engine_cash_flows`. Run Plan 05 AT-SIM-1 there.
5. Complete the simulation report and G6 sign-off for the pinned release bundle. For G7,
   create the simulation realm from a recorded snapshot of all active live-realm committee
   seats. Admit only copied `user` seats to the release vote. Before opening the proposal,
   require the simulation `PolicyConfiguration` to name the same design versions and
   calculation bundle as the proposal's release bundle, using the Plan 02 §15.4 G7 form of
   INV-16. Apply the release lock and require
   the frozen eligible voters to equal the active live-realm seats at the test time. Restart
   the G7 setup if the copied committee no longer matches. Run the vote in test mode with
   the owner's separate consent. Execute it by writing only
   `PolicyConfiguration.test_release` on the simulation configuration in one S-1 transaction.
   Record the release bundle and proposal ID in the G7 sign-off. Retain the seat snapshot
   as evidence for the same-bundle check in Plan 02 §15.4. Verify that G7 leaves the project
   lifecycle at its pre-execution state and that live settlement is disabled.

Adopt Plan 02's v1 `member_cover_set` adapter until Plan 05 accepts v2 and the realm-aware
snapshot call (C-1 and C-2). Keep the realm and exact cutoff in the trusted job context even
when adapting a v1 payload. Plan 04 owns voting mechanics, Plan 06 owns rulebook content and AI
opinion policy, and Plan 07 owns posting rules. Use fixture adapters to exercise architecture
before those plans are implemented. Shared G7 delivery uses their scoped implementations.

## 9. Milestones and acceptance

| Milestone | Deliverable and exit evidence |
|---|---|
| A0 — contracts | Record the assumptions in §10 and agree the Phase 1 interfaces with 02 and 05. Pick the fixture and align schemas, clock, key fixtures and test vectors. |
| A1 — platform foundation | Sign-in, tenant, project, current memberships, role enforcement, database-backed core and audit. Pass Plan 02's real parallel-write and failure injection cases for the delivered commands. Prove isolated tenants and realms, replay safety, hash-chain continuity, and consistent cross-aggregate snapshots under concurrent writes and rollback. |
| A2 — artifacts and jobs | Prove immutable conditional publication, exact-retry recovery, scope and realm binding, stale register-head conflict, worker fencing, cancellation and checkpoint recovery. Pass Plan 05 sandbox and determinism checks on two worker hosts. Prove only the scoped input manifest is accessible and output validation rejects invalid files. |
| A3 — first integration | Run §8 through the dummy journal with scoped fixtures for unfinished domain adapters. Kill a worker after register CAS and after journal commit. Resume and show the same trace, payout, journal and balance. Run equivalent concurrency and failure cases from a clean seed. |
| A4 — shared simulation | Replace fixture governance, rulebook and ledger adapters on the acceptance path. Complete G1–G7, verify the live realm and payment boundary, and rehearse the shared environment restore and erasure path. Record run IDs, digest and audit evidence, fixture balances and delivered conformance results. |

## 10. Decisions still needed

| Decision | Owner and deadline |
|---|---|
| Identity provider and hosting region | Platform engineering before A1 shared deployment. Confirm MFA, staff account lifecycle and data location. |
| Numerical versions, runner image and worker CPU class | Engine and platform engineering before A2. Record the exact compatibility and determinism evidence. |
| Backup targets, retention and data policy | Product and platform engineering before A4. Confirm test-environment targets and constrain staff data. Product and legal own the live policy and requirements before Phase 2. |
| Plan 05 handoff | Domain and engine owners before A3. Close Phase 1 C-7, C-8 and C-10 and agree register scoping and adapter acceptance for C-1/C-2. |
| Live pilot details | Owner and legal before Phase 2. Product, jurisdiction, custody, payments and licensing remain Plan 01 and Plan 05 G8 decisions. |
