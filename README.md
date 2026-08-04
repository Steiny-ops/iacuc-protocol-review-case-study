# IACUC Protocol Review System

A case study: building a multi-role research compliance workflow system in Google AppSheet, from zero platform experience to a feature-complete, twice-verified system in one week, then hardening it over the following months into something a committee could run on.

This repository documents the architecture, security model, and engineering lessons. It contains no institutional data, no real protocols, and no operational configuration. Names, identifiers, endpoints, vendors, and institution-specific details are omitted or genericized.

## The problem

A university research compliance office reviews animal-use protocols through an IACUC (Institutional Animal Care and Use Committee). The existing process ran on a legacy web platform, email threads, and human memory: reviewer assignments tracked by hand, revision rounds living in inboxes, statuses updated when someone remembered, and no enforced separation between what a researcher, a reviewer, and a committee chair should each see.

Constraints: one builder (a coordinator, not a developer), no budget, no development team, and the system had to run on tools the office already had, Google Workspace.

## The architecture

Four layers. The first three are deliberately boring, which is the point; the fourth is where the interesting failure modes live:

- **Database:** a Google Sheet. One tab per table, stable UUID keys, references by ID. At a compliance office's scale (roughly a hundred protocols a year), a Sheet is a perfectly good database, and it gives you version history, trivial backup, and a data layer the office can read without the app.
- **Application:** Google AppSheet over the Sheet. Tables, role-based security filters, action conditions, format rules, and automation bots. AppSheet is the interface and rules layer; the Sheet stays the truth.
- **Automation:** Google Apps Script attached to the Sheet for the jobs AppSheet cannot do: intake handling, a nightly timestamped backup, document filing and sweeps, a scheduled health monitor, a weekly workflow report, an append-only audit log, and a self-healing routine that keeps a load-bearing derived column populated (more below).
- **Intake:** a hosted submission page posting to a script Web App endpoint, which calls an edge worker that holds every third-party credential and performs document extraction. Described in its own section below.

Tables: Protocols, Users, Assignments, Reviews, RevisionRequests, PIResponses, ReviewerComments, CoordinatorNotes, plus an append-only AuditLog and a snapshot table supporting it. Users drives everything: a Role column that every security filter and action condition looks up by the signed-in email. Renewals and modifications are modeled without a new table: a protocol carries a submission type and an optional self-reference to its parent, so each renewal or modification is its own record with its own review lifecycle, linked to the original, which is never edited once approved and which keeps its identifier for the life of the family.

## The role model

Seven roles, very different systems:

| Role | Sees | Can do |
|---|---|---|
| Coordinator | Everything | Prepares intake, maintains all records, verifies the file before approval, keeps working notes; drives the lifecycle without deciding anything in it |
| Reviewer | Only assigned protocols, with co-reviewers' reviews and the researcher's responses | Writes and signs reviews; drafts and sends revision requests; posts committee-internal comments |
| Researcher (PI) | Only their own protocols; only revision requests actually sent to them | Submits immutable responses; does not edit the protocol record at all |
| Chair | Everything | Classifies each new protocol and assigns its reviewers at the front of the workflow, and approves the protocol at the end; alters nothing between those two points |
| Oversight | Everything, including coordinator working notes and committee comments | Nothing; view-only across the entire system |
| Veterinarian | Protocols flagged for veterinary oversight | Records a determination on flagged protocols; nothing else |
| Facility Manager | Protocols flagged for facility oversight | Records a determination on flagged protocols; nothing else |

Role is a single-value enum: one account holds exactly one role. That is a real constraint rather than an oversight, and it enforces separation of duties in the same spirit as the conflict-of-interest recusal below. It also means a person who genuinely holds two functions cannot be modeled without a schema change, which is a live design question rather than a solved one. The lesson on enum comparisons explains why widening it is not a one-dropdown change.

Visibility is enforced by security filters (rows a role cannot see are never loaded to their device, data-level privacy, not hidden UI). Mutation is enforced separately by action conditions and column-level editability. Keeping those two layers distinct was one of the central lessons: **visibility rules and mutation rules fail independently, and auditing one tells you nothing about the other.** Adding the view-only Oversight role made this concrete: granting it read access to every table exposed a set of note-management actions that had never carried role conditions, because they had relied on a security filter to stay hidden. View-only is not achieved by withholding views; it is achieved by withholding every mutating action, which must be verified action by action.

Conflict of interest is enforced in the same layer. Where the chair is also the researcher on a protocol, the chair actions are removed from that record by condition, so the recusal is structural rather than procedural.

## The confidentiality model

Reviewer identity is confidential from researchers, by committee direction rather than by design preference, and that requirement shapes several parts of the system:

- Researchers never load review records at all. Not hidden, not filtered in the interface, simply absent from their data universe.
- The only reviewer artifact a researcher ever receives is a sent revision request, and on it the reviewer is identified by a generic sequential label rather than a name.
- Committee-internal comments are a separate table with its own filter: reviewers on the protocol, the coordinator, the chair, and oversight can read them; researchers cannot.
- The masking is implemented as display columns that resolve differently by role, which is deliberately more complex than a hidden field and must not be simplified back into one.

## The integrity model

The system's design center is that a compliance record must be trustworthy, which produced five rules:

1. **Signed reviews are permanent.** Signing is a deliberate, confirmed action; once signed, no role, including coordinators, can edit or delete the review through the app. The rare correction is a data-layer act, noted in the record.
2. **Researcher responses are immutable.** Written only by the researcher, unchangeable by anyone once submitted. A wrong response is corrected by a new response, preserving the original.
3. **Reviewed content is stable.** The researcher does not edit the protocol record; all edits are made by a coordinator (or by the chair during the front-end review step). Content does not shift under a reviewer mid-review, so signatures always refer to the text that was actually reviewed.
4. **Drafts are private and inert.** A revision request is an editable private draft until the reviewer deliberately sends it; an unsent draft never emails anyone, never changes a status, and is never visible to the researcher.
5. **Revision requests are reviewer-only.** A coordinator cannot issue one on a reviewer's behalf. The request is the reviewer's instrument, and the audit record should never show the office speaking in a reviewer's voice.

## The lifecycle and automation

A protocol begins under chair review: every new record starts in a chair-review state where the chair classifies it (identifier, review type, oversight flags) and assigns its reviewers, then advances it into review with a single confirmed action. From there, statuses are earned, not remembered:

- Sending a revision request advances a protocol to Revisions Requested.
- Signing a review, resolving a revision request, or recording an oversight determination triggers a completion check: when every review is signed and every flagged determination is recorded, whichever finishes last, the protocol advances itself to the chair-approval status.
- A revision request sent after completion automatically reopens the protocol.
- At the end, the chair approves the protocol with a confirmed action that stamps an approval timestamp and an expiration exactly one year ahead, then locks the record's status. Ending an approved protocol early is a separate, deliberate coordinator action that records a termination timestamp and freezes the record.

The completion gate is worth describing precisely, because it is the part most likely to be modeled wrongly. It is not one condition but several independent branches (reviews, veterinary determination, facility determination), each of which fires the same check, so whichever branch completes last is the one that advances the protocol. Branches that are not required for a given protocol are satisfied trivially. The failure mode this design invites is a branch that can never be satisfied, for example an oversight flag set on a record that no eligible account is assigned to, which stalls a protocol silently rather than erroring.

The completion and reopen transitions run as platform bots with transition-safe conditions; the chair-review advance, the approval, and the termination run as explicit role-gated actions rather than bots, because each represents a human decision that should be taken deliberately rather than triggered as a side effect. (An earlier design advanced a protocol automatically the moment a reviewer was assigned; once the chair-review step was added at the front, that automatic advance was retired, because assignment now happens while the chair is still working and must not move the protocol on its own.)

One derived column sits outside the bots and is worth calling out, because it is where the platform's silent-failure tendency bit hardest. Cross-row visibility (a reviewer seeing co-reviewers on a shared protocol) is driven by a real column that aggregates reviewer emails per protocol, computed by a Sheet formula rather than an in-app formula, because a security filter is evaluated at sync time and an in-app formula would go stale the instant a different table changed. A Sheet formula, though, can be silently erased by a data wipe or never written onto a script-created row. The fix is a self-healing script function: it walks the column, leaves correct cells alone, and rewrites the formula into any cell missing it, running on every intake and nightly. Building it surfaced a latent bug in the formula itself (a row-anchored range that would have silently dropped data), an instance of a safeguard auditing the thing it protects before it was ever needed.

Every bot uses transition-safe conditions (`AND([Flag] = TRUE, [_THISROW_BEFORE].[Flag] <> TRUE)`) and listens to both row creation and row updates, for reasons documented in the lessons file.

## The intake pipeline

Researchers submit through a separate institutional application portal, not into the workflow system directly, which means intake is an integration problem rather than a form problem. The pipeline is deliberately layered so that no credential and no model call ever sits in the browser:

1. A hosted submission page collects the document and posts to a script Web App endpoint. The page holds no secrets; the endpoint is gated by a token and accepts requests only from the page's own origin.
2. The endpoint calls an edge worker that holds every third-party credential. The worker's privileged routes are guarded by a shared secret held in both the worker's environment and the script's properties, so the endpoint can reach the worker but a stranger holding the page's token cannot.
3. The worker performs document extraction with a hosted language model and returns structured fields. The extraction reads the submitted document as a document, because form layout is a large part of what makes the extraction accurate, which turned out to constrain vendor choice more than model quality did.
4. The endpoint creates the protocol record, files supporting documents, advances a sequence counter, and creates a tracking card on the office's project board.

Two consequences worth recording. First, records created this way are written to the Sheet directly, so no platform bot fires on them: the intake code must send its own notifications, and a status that would normally be earned has to be set explicitly. Second, an integration that sends documents to a language model is a compliance question in its own right at an institution with vendor approval requirements, and the argument that survives review is about capability and attestability (can you show which model version read a document, and can you validate against that version) rather than preference. A router that silently selects among models cannot be attested to, whatever its output quality.

## The operations layer

The office runs the system without a developer on call, which meant the system had to report on itself:

- A nightly backup writes a timestamped copy of the entire data layer to a fixed destination, pinned by identifier rather than by name, because two folders can share a name and the wrong one will accept writes silently.
- A scheduled health monitor runs roughly twenty checks over the data and the integrations, including a read-only heartbeat against the edge worker that distinguishes a missing secret from an unreachable service from an authentication drift.
- A weekly report summarizes the week's movement for the coordinator and lists what needs attention, including expirations far enough ahead to act on.
- A weekly digest goes to reviewers with outstanding work, so an unsigned review surfaces without the coordinator chasing it.
- An append-only audit log records every field-level change with actor and timestamp, supported by a snapshot table that lets a scheduled job diff state and detect changes made outside the app.

The audit log's honest limitation is documented in the file itself: count-and-tail-hash stamping detects appends and truncation, not a mid-log edit. A chained-hash design is the fix, and it is deferred rather than pretended.

## Testing

Verification happened in two distinct modes, and they found different classes of defect.

**Adversarial role-switching**, run by the builder: run the entire lifecycle, intake through final status, as each role in turn, and try to do things each role should not be able to do. Two full end-to-end passes on clean data. This method, not unit-level checks, found every significant permission and automation defect: a security filter that silently blocked legitimate users, a status guard that silently swallowed a legitimate transition, a bot that missed flag-at-creation events, and a permission model where visibility had outrun mutation control.

**Multi-person user testing**, run with real committee members and researchers on seeded data, each working from a role-specific packet with a written checklist, with the relay-critical steps sequenced so one person's output was the next person's input. The full workflow relayed end to end across four people on the first attempt.

The two modes are not interchangeable, and the difference is the most useful testing lesson from the project. **Testers exercising a system find discoverability problems**: a control they could not locate, a screen that hid the context they needed, a step whose order was unclear. **Domain experts looking at a system find correctness problems**: a workflow that does not match how the committee actually decides. Round one produced mostly the former from testers and the latter from the chair, which is an argument for giving domain experts review tasks rather than only use tasks.

Two structural findings are worth naming because both were invisible to single-path testing. A completion check that examines only the record that triggered it will behave incorrectly when a protocol carries several outstanding items of the same kind, and that only surfaces if the test data includes a protocol with two of them from two different people. And a generated export inherits none of the application's confidentiality rules: a document assembled for distribution was found to carry precisely what the interface withholds. Nothing propagates an access model into an export template, so every generated artifact needs its own audit against that model, and this one is open rather than solved.

One role went untested, and the reason is structural rather than an oversight in planning. The builder occupies an operational role in the office, and a checklist the builder passes on their own role is not evidence: it leaves the largest role untested while appearing tested. That verification waits for a second person, and the gap is recorded as a gap.

The dominant failure mode of low-code platforms is **silence**: misconfigurations do not error, they quietly do nothing. The countermeasures are procedural, a smoke-test ritual after structural changes, and a maintenance reference that records not just every rule but the reasoning behind it, so the next maintainer can tell deliberate from accidental.

## Results

One week from first login to: a functionally complete multi-role workflow covering the full protocol lifecycle including renewals and modifications, an automation chain, a self-healing data layer, a layered integrity model adopted by the office as policy, two full adversarial verifications, and a maintained document set (user guide, governance rules, build plan, technical reference). The week also included a deliberate data wipe and recovery, a root-caused regression, and several design reversals absorbed cleanly, the operational history most projects only accumulate after launch.

The months since added the parts that make a system survivable: a real intake pipeline, an operations layer that reports on itself, an audit trail, a confidentiality model with committee force behind it, a multi-person testing round, and a documentation set covering every role.

The evolution is worth reading as a list, because most of it is the kind of work that only appears after a system meets its users. A chair-review step was added at the front of the workflow, so the committee chair classifies each protocol and assigns its reviewers before review begins; a chair-approval step and a termination step were added at the end, giving the record a complete lifecycle from intake through approval and, where needed, early termination, each with its own stamped timestamps and record-locking. A view-only Oversight role was added for management, which prompted a full audit confirming that read access and mutation are governed independently. Veterinary and facility oversight became first-class roles with their own determination step and their own branch of the completion gate. Intake moved from a form to a document-extraction pipeline. The identifier scheme gained a review-type suffix assigned by the chair at classification. A handful of now-unused early constructs were retired, and a one-year expiration calculation was corrected from a fixed-day offset to an exact same-date-next-year computation so it stays accurate across leap years. Expiration reporting was made supersede-aware, so a superseded parent record stops generating noise while a genuinely lapsed protocol still surfaces.

Not everything is closed, and the open items are as instructive as the closed ones: a single-role user model that a genuinely dual-role person cannot fit, an audit log that cannot yet detect a mid-log edit, an export surface that does not inherit the confidentiality model, and a role separation enforced at assignment time rather than at access time.

See [docs/lessons-learned.md](docs/lessons-learned.md) for the engineering lessons, which are the genuinely transferable output.

## Author

Built and documented by Spencer Steinberg. This case study is personal work product describing generalizable architecture and lessons; it intentionally contains no institutional information.
