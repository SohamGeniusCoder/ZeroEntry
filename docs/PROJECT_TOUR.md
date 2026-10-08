# ZeroEntry: a project tour

This guide explains the **implemented ZE2 prototype**, not every historical proposal.
Use the [README](../README.md) to run it, the [demo script](development/DEMO_SCRIPT.md) to present it, and
[PROJECT_SPEC](../PROJECT_SPEC.md) for authoritative rules.

## The one-minute explanation

ZeroEntry is a database-first sanitation-safety and work-evidence accountability system.

It asks three different questions:

1. **Before work:** has mechanised clearance been attempted, and is this exceptional workflow eligible even to proceed?
2. **During a simulated exceptional entry:** does every required person, resource and current evidence record satisfy the gate?
3. **After closure:** is the claimed completion supported by the records that should exist, and what needs a human review?

The first goal is mechanised work with no human entry. A green educational permit is not the project goal and is not
a real-world legal clearance. PostgreSQL owns the recorded decisions; the UI makes their reasons inspectable.

## 1. Complaint, location and job

A complaint refers to a manhole within an urban local body (ULB). A job links work on that complaint to a contractor.
The contractor and instruments must be governed within the relevant scope.

**Why separate these entities?** A complaint can have more than one job and work attempt. Putting everything into one
complaint row would duplicate contractor, machine, crew and invoice information and make historical relationships ambiguous.
Foreign keys preserve real links; relational modeling supports accountability without assuming every record has every link.

## 2. Mechanised-first evidence

An engineer records machine deployments and their outcomes. A machine clearance is one route to accountable completion.
An exceptional permit requires a written engineer-approved waiver; a failed-machine justification needs recorded failure.

**Why isn't a waiver enough?** Operational justification and jurisdiction eligibility answer different questions.
Real or unknown policy defaults to review/denial. Only clearly labelled educational scopes allow the positive software simulation.
A waiver cannot turn a prohibited or unknown real workflow into a permitted one.

## 3. Build the draft evidence set

The supervisor prepares a DRAFT permit. It includes:

- Distinct entrant, standby and supervisor roles.
- Each assigned person's own acknowledgment of participation.
- Current medical/training evidence and contractor credentials.
- Required personal equipment and inspected serial assets.
- Separate shared site equipment and exclusive resource allocation.
- Seven typed readiness areas: structure, isolation, ventilation, rescue, communication, traffic and medical.
- O2, H2S, LEL and CO observations at TOP, MID and BOTTOM, including time, detector and provenance.

**Why acknowledgment?** Assignment is an administrative plan, not proof that the person actually accepted participation.

**Why serial assets and shared equipment?** A catalogue row is not an available inspected device. One rescue asset or
crew member must not satisfy two simultaneous permits. Personal PPE alone does not establish site readiness.

**Why typed/unknown/simulated gas values?** Missing is not zero, and a typed value is not authenticated hardware truth.
Provenance makes those limitations visible rather than decorating an unverifiable reading with a "safe" label.

## 4. Ask the database for a decision

The authenticated API opens a transaction, supplies transaction-local role/scope context and calls database routines.
PostgreSQL checks current policy and evidence under locks. Relational division implements the "every required item/person/depth"
question; other constraints, functions and triggers enforce relationships and state changes.

Any failing clause denies the decision. Both positive and negative decisions retain receipts. A successful authorization
also preserves its original historical snapshot. Receipts bind the decision to an evidence revision and expiry.

**Why the database?** Multiple clients can write to the same system. A UI-only or Python-only check would not protect every
runtime write path. Database guards also protect a raw runtime UPDATE. The privileged database owner remains trusted.

**Why not trust the preview?** Evidence and resource availability can change between reading a checklist and committing a
decision. Locking and admission rechecks address this time-of-check/time-of-use gap. A properly locked application can also
address such a race; our comparison does not claim otherwise.

## 5. Admission is a new decision

Authorization is not permission to admit people forever. Opening an entry log requires a current passing gate, the right
crew role, the configured time window, non-overlap and rest constraints. The entry is linked to the actual crew member.

**Why retain a historical receipt but recheck now?** Historical explanation and present eligibility are different data.
Aging cannot rewrite what was known at an earlier decision; it can stop that decision being used for a new admission.

The configured stretch/rest and daylight rules have documented source/applicability notes. They are not presented as a
complete field checklist or a substitute for a site operator's responsibilities.

## 6. Adverse evidence stops permission, not the record of a person

New unsafe gas, revoked prerequisites or expiry can stop an active permit. The maintenance loop also checks time-driven
changes periodically. ABORTED is terminal; recovery needs a new permit.

An already-open physical interval remains recorded until an exit is entered. Exits remain allowed after stop/expiry and
overrun evidence is preserved. Resource occupancy is not released merely because permission stopped.

**Why?** "This person must not enter" and "this person is now outside" are not equivalent statements. Deleting occupancy or
refusing truthful exits would make the database less accurate at exactly the time it matters.

**Limit:** this software has no independent sensor or physical gate, and API/database outages delay periodic sweeps.
It governs stored records, not physical rescue or prevention.

## 7. Check closure against its expected evidence

The SE1 detector starts from resolved complaints whose resolution type requires evidence, including complaints with no job.
It expects accountable machine clearance or a suitably closed/logged permit.

SE2 checks whether every expected entrant on a closed permit has an entry record. Exempt resolution types deliberately
do not require clearance evidence. The configured grace window handles recording delays.

Machine final-outcome **server receipt time** is separate from event time. Otherwise, an old event timestamp on a late
upload could falsely make evidence look timely.

**Why absence detection?** A simple list of existing permit records cannot reveal complaints that have no permit or job.
Starting from the explicit expected population and anti-joining evidence exposes reviewable gaps.

**What it does not prove:** absence does not establish that an entry occurred or that anyone committed misconduct.

## 8. Imported claims and late uploads stay reviewable

An external completion claim is not silently equated with internal proof. It can be unmatched, explicitly matched or
ambiguous. Completion projections are recomputable; their live relational reference remains authoritative.

Late evidence does not automatically erase an alert. Review history records the human disposition.
Repeated detector scans are idempotent rather than creating unlimited duplicate alerts.

**Why?** A claim, evidence, inference and reviewer conclusion must remain distinguishable. Otherwise a later edit could
silently remove the discrepancy that prompted oversight.

## 9. Financial consequences preserve their source

An invoice hold identifies the alert, registered incident or standalone report that created it.
Dismissing one alert can release only that alert's hold. An unrelated incident hold remains.

Locks coordinate hold creation with approval/payment, including concurrent decisions.
Legacy registered-incident consequences run atomically: their configured internal cases/statuses/holds commit together or roll back.

**Why provenance?** A single held=true flag cannot explain multiple independent reasons or safely release just one.
These prototype decisions are not bank transfers, external sanctions or automatically established legal entitlements.

## 10. A victim does not need an existing registry identity

Standalone incident reports support multiple victims and optional worker/job/permit/contractor links.
Each victim can receive a pending internal assessment reference.

**Why separate report, victim and assessment?** One event can affect multiple people. Unregistered people must not be excluded
by mandatory foreign keys to a worker registry, nor should the application invent identities or pretend an assessment is paid.

The original registered incident workflow remains available; the new intake extends coverage without falsifying older data.

## 11. Retries, notifications and security

Mutating commands use exact idempotency keys scoped to actor and operation, with request-body hashes and retained responses.
A dropped-response retry reuses the same key. A changed body with the same key conflicts rather than creating ambiguous state.

Notifications come from a transactionally stored, scope-ordered outbox. The browser can replay committed events after
refresh; a transient network message is not treated as evidence.

Opaque revocable sessions, cookie CSRF checks, server-side role guards, least-privilege SQL and row-level security establish
access boundaries. Worker/contractor scope comes from the server session, not a client-selected role.

**Why plain JavaScript instead of a new framework?** The existing no-build UI already supported the working stack.
Keeping it preserved teammate work and avoided a rewrite. The integrated operations-board styling adds responsive light/dark
layout and accessible tabs while retaining policy, receipt, reconciliation and incident workflows.

## 12. What the tests and evaluation actually show

The final source-bound local report records **620 passed**, including **six Chromium browser scenarios**.
Tests cover real PostgreSQL constraints/triggers, transactions and races, role/scoped API behavior, retries and selected UI flows.

The DBMS demonstration includes all six lab areas through actual migrations, queries, functions, procedures, triggers and a cursor.
The [submission map](SUBMISSION.md) identifies artifacts rather than treating table count as technical depth.

Synthetic evaluation studies explicit baselines and known gaps. Some comparisons intentionally have different semantics;
the unguarded database ablation is not a fair claim against every possible application validator.
Logical restore checks are not crash-recovery or production certification.

Always distinguish **local tests**, **synthetic benchmark results**, **remote CI**, and **field outcomes**.
Only the first two are recorded here as prototype evidence.

## Presentation closing

"Our novelty is the integration: mechanised-first work, actual participation, current shared-resource and readiness checks,
revision-bound admission, reviewable closure gaps, source-linked holds and intake for unregistered victims—all backed by
relational integrity and transactional evidence. We validate software behavior on synthetic data, not real-world safety."

For the exact live order, IDs and timing, use the [current demo script](development/DEMO_SCRIPT.md).
