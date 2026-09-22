# IACUC Protocol Review System

A case study: building a multi-role research compliance workflow system in Google AppSheet, from zero platform experience to a feature-complete, twice-verified system in one week, then hardening it over the following months into something a committee could run on.

**Status, September 2026:** built, verified through adversarial testing and two rounds of user testing, reviewed for accessibility, and handed to a departmental account. Pending launch; the system has run on demonstration data throughout and has not yet processed a live protocol.

This repository documents the architecture, security model, and engineering lessons. It contains no institutional data, no real protocols, and no operational configuration. Names, identifiers, endpoints, vendors, and institution-specific details are omitted or genericized.

## The problem

A university research compliance office reviews animal-use protocols through an IACUC (Institutional Animal Care and Use Committee). The existing process ran on a legacy web platform, email threads, and human memory: reviewer assignments tracked by hand, revision rounds living in inboxes, statuses updated when someone remembered, and no enforced separation between what a researcher, a reviewer, and a committee chair should each see.

Constraints: one builder (a coordinator, not a developer), no budget, no development team, and the system had to run on tools the office already had, Google Workspace.

## The architecture

Four layers. The first three are deliberately boring, which is the point; the fourth is where the interesting failure modes live:

- **Database:** a Google Sheet. One tab per table, stable UUID keys, references by ID. At a compliance office's scale (roughly a hundred protocols a year), a Sheet is a perfectly good database, and it gives you version history, trivial backup, and a data layer the office can read without the app.
- **Application:** Google AppSheet over the Sheet. Tables, role-based security filters, action conditions, format rules, and automation bots. AppSheet is the interface and rules layer; the Sheet stays the truth.
- **Automation:** Google Apps Script bound to the Sheet for the jobs AppSheet cannot do: intake handling, a nightly timestamped backup, a document-filing sweep, a scheduled health monitor, a weekly workflow report, an append-only audit log, and a self-healing routine that keeps a load-bearing derived column populated (more below).
- **Intake:** a hosted submission page posting to a script Web App endpoint, which calls an edge worker that holds every third-party credential and performs document extraction. Described in its own section below.

Tables: Protocols, Users, Assignments, Reviews, RevisionRequests, PIResponses, ReviewerComments, CoordinatorNotes, a small information table behind the in-app information page, plus an append-only AuditLog and a snapshot table supporting it. Users drives everything: a Role column that every security filter and action condition looks up by the signed-in email.

Renewals and modifications are modeled without a new table: a protocol carries a submission type and an optional self-reference to its parent, so each renewal or modification is its own record with its own review lifecycle, linked to the original, which is never edited once approved. The family shares one protocol identifier for its whole life, and that single decision turned out to be the source of an entire class of defects: the identifier looks like a key and is not one. Four separate defects expressed the same wrong assumption before the pattern was recognized, in a document-filing routine, the intake handler, a chair action, and a filename scheme where a parent and its renewal generating on the same day produced byte-identical files. Fixing the instances was not enough; the fix was a sweep of every script and every expression that used the identifier to find or distinguish anything, checked against a family of at least two records.

Revision requests carry the same kind of self-reference: a follow-up request points to the request it follows, so a chain of requests is recorded as linked records rather than as edits to one.

## The role model

Seven roles, very different systems:

| Role | Sees | Can do |
|---|---|---|
| Coordinator | Everything | Prepares intake, admits researchers to the application, maintains all records, confirms training and clearance before approval, keeps working notes; drives the lifecycle without deciding anything in it |
| Reviewer | Only assigned protocols, with co-reviewers' reviews and the researcher's responses; only their own revision requests | Writes and signs reviews; drafts and sends revision requests and follow-ups; posts committee-internal comments |
| Researcher (PI) | Only their own protocols; only revision requests actually sent to them | Submits immutable responses; does not edit the protocol record at all |
| Chair | Everything | Classifies each new protocol and assigns its reviewers at the front of the workflow, records an outcome on renewals and modifications, may pause or withdraw a protocol before approval, and approves at the end |
| Oversight | Everything, including coordinator working notes and committee comments | Nothing; view-only across the entire system |
| Veterinarian | Protocols they are assigned to for veterinary oversight | Records a determination on those protocols; nothing else |
| Facility Manager | Protocols they are assigned to for facility oversight | Records a determination on those protocols; nothing else |

Visibility is enforced by security filters (rows a role cannot see are never loaded to their device, data-level privacy, not hidden UI). Mutation is enforced separately by action conditions and column-level editability. Keeping those two layers distinct was one of the central lessons: **visibility rules and mutation rules fail independently, and auditing one tells you nothing about the other.** Adding the view-only Oversight role made this concrete: granting it read access to every table exposed a set of note-management actions that had never carried role conditions, because they had relied on a security filter to stay hidden. View-only is not achieved by withholding views; it is achieved by withholding every mutating action, which must be verified action by action.

Conflict of interest is enforced in the same layer. Where the chair is also the researcher on a protocol, the chair actions are removed from that record by condition, so the recusal is structural rather than procedural.

**Role is a single-value enum:** one account holds exactly one role. That enforces separation of duties in the same spirit as the recusal, and it means a person who genuinely holds two functions cannot simply be given two roles. Widening it is not a one-dropdown change, because every comparison written as `[Role] = "X"` silently returns false against a list-valued column; nothing errors, filters just go empty.

The real case that forced the question was a reviewer who is also the right person to speak for the facility. The resolution was to stop asking the Role column the question at all. **Facility oversight is now gated by assignment rather than by role:** whoever is named on the protocol as facility manager can see it and record the determination, and the assignment picker offers facility managers and reviewers. **Veterinary oversight deliberately stays gated by role,** because a reviewer will never hold a veterinary credential and the role check is the only thing in the system ensuring the veterinarian of record actually is one. The two sides look inconsistent and are documented as intentional, so no future maintainer tidies them into symmetry. The diagnostic that shaped the change is worth copying: before widening a picker, check whether the downstream screens and actions test the assignment or the role. Here the facility action was already assignment-gated while the security filter was role-gated, so a reviewer assigned as facility manager on a protocol they were also reviewing got in by accident through their reviewer assignment, and the same person on a protocol they were not reviewing saw nothing, silently.

One consequence is recorded as a committee question rather than a configuration one: a single person can now supply both a reviewer's judgment and the facility determination on the same protocol, and the completion gate counts both.

## The confidentiality model

Reviewer identity is confidential from researchers, by committee direction rather than by design preference, and that requirement shapes several parts of the system:

- Researchers never load review records at all. Not hidden, not filtered in the interface, simply absent from their data universe.
- The only reviewer artifact a researcher ever receives is a sent revision request, and on it the reviewer is identified by a generic sequential label rather than a name.
- Committee-internal comments are a separate table with its own filter: reviewers on the protocol, the coordinator, the chair, and oversight can read them; researchers cannot.
- The masking is implemented as display columns that resolve differently by role, using an allow-list of entitled roles rather than a deny-list of the researcher role. A deny-list fails open: an account missing from the users table has no role, does not match the researcher test, and would be shown the name. The allow-list fails closed. It is deliberately more complex than a hidden field and must not be simplified back into one.
- Notifications follow the same rule. When a protocol is approved or withdrawn, the committee side is told by a separate email from the one the researcher receives, because putting reviewers on the researcher's message would name them.

**Generated documents were the hole, and the fix was a second document rather than a mask.** A PDF assembled for distribution inherits none of the application's access rules: the first version carried exactly what the interface withholds. The system now produces two documents. The summary is the committee's internal record, carrying reviewer names and the full review correspondence, and it stays inside the committee. The protocol record is the shareable one, and it omits reviewer identities and correspondence entirely rather than redacting them. Nothing propagates an access model into an export template, so every generated artifact needs its own audit against that model.

## The integrity model

The system's design center is that a compliance record must be trustworthy, which produced five rules:

1. **Signed reviews are permanent.** Signing is a deliberate, confirmed action; once signed, no role, including coordinators, can edit or delete the review through the app. The rare correction is a data-layer act, noted in the record.
2. **Researcher responses are immutable.** Written only by the researcher, unchangeable by anyone once submitted. A wrong response is corrected by a new response, preserving the original.
3. **Reviewed content is stable.** The researcher does not edit the protocol record; all edits are made by a coordinator (or by the chair during the front-end review step). Content does not shift under a reviewer mid-review, so signatures always refer to the text that was actually reviewed.
4. **Drafts are private and inert.** A revision request is an editable private draft until the reviewer deliberately sends it; an unsent draft never emails anyone, never changes a status, and is never visible to the researcher. Once sent, it locks.
5. **Revision requests are reviewer-only.** A coordinator cannot issue one on a reviewer's behalf. The request is the reviewer's instrument, and the audit record should never show the office speaking in a reviewer's voice. A follow-up is a new request linked to the original, never an edit to it, so the record shows the sequence of the conversation rather than its final state.

## The lifecycle and automation

Eleven statuses: eight on the main path and three off it. Statuses are earned, not remembered.

**Intake.** A protocol starts at a coordinator step. Four things must be present before it can leave: the identifier, the document folder, a confirmation that required training is verified, and a confirmation that the researcher has been admitted to the application. That last one exists because the platform has two independent gates: a manually kept allow list decides who can open the application at all, and the security filters decide what they see once inside. A row in the users table grants nothing at the door, so a researcher who is in the table but not on the allow list signs in successfully and is then refused with a message that looks exactly like a licensing error. The confirmation field makes admitting the researcher a recorded step, and a pair of stamp columns records who confirmed it and when, because the audit log cannot attribute that particular change reliably (see the operations section).

**Chair review.** The chair classifies the protocol (review level, species coverage, veterinary and facility oversight, whether a medical clearance is required, whether it needs full-committee review) and assigns its reviewers, then advances it into review with a single confirmed action. On a renewal or modification the chair has three outcomes instead of one: send it on, return it for revision, or determine that a new protocol is required.

**Review.**

- Sending a revision request moves the protocol to Revisions Requested. It returns to review only when every outstanding request is resolved, not merely the one just closed.
- Where a researcher's answer does not settle a concern, the reviewer raises a follow-up from the original request instead of resolving it. The two are separate records, resolved independently, and the chain shows in the interface and in the generated summary.
- When every assigned review is signed, every required oversight determination is recorded, and no revision request is open, the protocol advances itself to a final coordinator check, where training and any required medical clearance are confirmed before the coordinator sends it to the chair.
- Once a protocol leaves the review stage, reviewers can no longer issue requests against it.

**Approval and after.** The chair approves with a confirmed action that stamps an approval timestamp and an expiration exactly one year ahead, then locks the record's status. Ending an approved protocol early is a separate, deliberate coordinator action that records a termination timestamp and freezes the record.

**Off the path.** A protocol can be paused at the researcher's request and later resumed to the exact status it was paused from. It can be withdrawn before approval, which requires a written reason, notifies everyone who worked on it, and removes it from the working views while keeping its full history. And a renewal or modification can end in the chair's determination that a new protocol is required.

**The completion gate** is worth describing precisely, because it is the part most likely to be modeled wrongly. It is not one condition but several independent branches (reviews, veterinary determination, facility determination), each of which fires the same check, so whichever branch completes last advances the protocol. Branches that are not required for a given protocol are satisfied trivially. Two failure modes are worth knowing. A branch that can never be satisfied, for example an oversight flag on a protocol with no eligible assignee, stalls a protocol silently rather than erroring. And two branches that reach the same decision by different methods can drift apart: here the reviewer branch compares the set of assigned addresses against the set of signed reviewers, while the facility branch compares counts. When one table held addresses in a different form from another, the address comparison could never empty and the reviewer path stopped advancing protocols, while the count comparison carried on working. Nothing reported a failure. The fix was to the data, not the logic, but the lesson is about the design: a gate that compares identities depends on every table spelling an identity the same way.

The completion transition and the return from revisions to review run as platform bots with transition-safe conditions; the chair-review advance, the approval, the pause, the withdrawal, and the termination run as explicit role-gated actions rather than bots, because each represents a human decision that should be taken deliberately rather than triggered as a side effect. (An earlier design advanced a protocol automatically the moment a reviewer was assigned; once the chair-review step was added at the front, that automatic advance was retired, because assignment now happens while the chair is still working and must not move the protocol on its own.)

Three platform behaviors govern how the automation is built, and all three fail silently:

- **Bots do not trigger bots.** A data change made by a bot does not fire other bots' change events. Where a status is set by a bot, its notification must be an email step inside that same bot; a separate notification bot watching for the status will never fire.
- **A bot has no user.** A named action invoked by a bot runs with no signed-in identity, so any condition inside it that tests the current user's email or role evaluates against a blank and the action quietly declines to run. The bot's run log still reads Complete, every step listed and timestamped, while the data never changed. One instance was found while building the intake gate, and every bot in the app was then swept for others. The rule: a condition on a bot-invoked action may test the row, never the user.
- **Scheduled bots need to see filtered data.** A scheduled bot also runs with no user, so every security-filtered table looks empty to it unless it is explicitly set to bypass the filters. The weekly reviewer digest matched no one until that setting was found.

Every bot uses transition-safe conditions (`AND([Flag] = TRUE, [_THISROW_BEFORE].[Flag] <> TRUE)`) and listens to both row creation and row updates, for reasons documented in the lessons file.

One derived column sits outside the bots and is worth calling out, because it is where the platform's silent-failure tendency bit hardest. Cross-row visibility (a reviewer seeing co-reviewers on a shared protocol) is driven by a real column that aggregates reviewer emails per protocol, computed by a Sheet formula rather than an in-app formula, because a security filter is evaluated at sync time and an in-app formula would go stale the instant a different table changed. A Sheet formula, though, can be silently erased by a data wipe or never written onto a script-created row. The fix is a self-healing script function: it walks the column, leaves correct cells alone, and rewrites the formula into any cell missing it, running on every intake and nightly. Building it surfaced a latent bug in the formula itself (a row-anchored range that would have silently dropped data), an instance of a safeguard auditing the thing it protects before it was ever needed.

## The intake pipeline

Researchers submit through a separate institutional application portal, not into the workflow system directly, which means intake is an integration problem rather than a form problem. The pipeline is deliberately layered so that no credential and no model call ever sits in the browser:

1. A hosted submission page collects the document and posts to a script Web App endpoint. The page holds no secrets; the endpoint is gated by a token and accepts requests only from the page's own origin.
2. The endpoint calls an edge worker that holds every third-party credential. The worker's privileged routes are guarded by a shared secret held in both the worker's environment and the script's properties, so the endpoint can reach the worker but a stranger holding the page's token cannot.
3. The worker performs document extraction with a hosted language model and returns structured fields, which the coordinator confirms on screen before anything is saved. The extraction reads the submitted document as a document, because form layout is a large part of what makes the extraction accurate, which turned out to constrain vendor choice more than model quality did.
4. The endpoint creates the protocol record, files supporting documents, advances a sequence counter, and creates a tracking card on the office's project board. A renewal or modification inherits its parent's identifier rather than consuming a new one.

Three consequences worth recording. First, records created this way are written to the Sheet directly, so no platform bot fires on them: the intake code must send its own notifications, and a status that would normally be earned has to be set explicitly. Second, an integration that sends documents to a language model is a compliance question in its own right at an institution with vendor approval requirements, and the argument that survives review is about capability and attestability (can you show which model version read a document, and can you validate against that version) rather than preference. A router that silently selects among models cannot be attested to, whatever its output quality. Third, the governance rule that proved workable was not about whether the extraction step is switched on but about what data reaches it: the step may run while the system holds only demonstration data, and no live application reaches it until it runs on an institutionally approved provider under a departmental account with review complete. A rule that says a working step is off, while the office uses it, is a rule nobody is following.

## Notifications

Seventeen automatic notices cover the lifecycle, each sent at the point where action lands on someone: the chair when a protocol needs classifying, reviewers and any assigned overseers when review starts, the researcher when revisions are requested and a confirmation when their response is recorded, the reviewer when that response arrives, the coordinator when a protocol reaches its final check, the chair when it is ready for approval, and so on through approval, the off-path endings, and a weekly digest that sends each reviewer only their own pending items and sends nothing to people with nothing pending.

A few rules came out of building them:

- **Recipient lists are expressions, and the platform is strict about their types.** Lists combine by addition and every term must itself be a list, so a conditional single address is a one-item list or an empty list, never a bare value or an empty string. The same expression worked when entered through the expression editor and failed when pasted into the plain field, because the field treats its content as literal addresses unless it begins with an equals sign.
- **Copying a template body copies only the body.** New notices are built by copying an existing notice's full HTML and changing only the wording, which keeps them visually consistent. But the copy carries none of the other fields: two notices added late were found with no copy to the office mailbox and no reply-to address, because those live outside the body. They were set deliberately and verified on a live send.
- **Every template carries the same accessibility properties,** set after an external accessibility review: a fluid rather than fixed width, text contrast measured against the standard rather than chosen by eye, a minimum text size in the footer, layout tables marked as presentational so a screen reader does not announce them as data, and a line telling the reader how to get another format.

## Ownership and survivability

A system built by one person on that person's account is a system that leaves with them. The application, its scripts, and their scheduled triggers were transferred to a departmental account run by the office, and the data lives on an organization-owned shared drive that survives any individual account being deleted.

The transfer itself is worth recording, because it broke the application completely in a way that reads like the wrong problem. The platform supports a true owner-to-owner transfer that preserves the application's identity, so existing links in sent notifications kept working. Immediately afterwards, though, every table failed to load with an authentication-scope error, cascading from the users table because every security filter resolves through it. That is not a permission failure, and the two read differently: a permission problem says not found or access denied. Sharing every file and folder with the new account is necessary but not sufficient, because the application reaches its data through a per-account authorized data source, and a newly created account has none. Adding and authorizing one fixed it. Scheduled script triggers are per-user and were reinstalled under the new account, each by an installer that removes its own old triggers first so that running it twice cannot create duplicates.

Alarms and reports resolve their recipients from the users table by role rather than from a hardcoded address, so they follow whoever holds the role. That matters most for the health monitor, whose design principle is that its daily arrival is the proof it is alive: an alarm addressed to one person's account stops being an alarm the day that account goes away.

One integration still runs on a personal account, and it is recorded as the outstanding ownership item rather than smoothed over.

## The operations layer

The office runs the system without a developer on call, which meant the system had to report on itself:

- A nightly backup writes a timestamped copy of the entire data layer to a fixed destination and keeps ninety days of them. The destination is pinned by identifier rather than by name, because two folders can share a name and the wrong one will accept writes silently. The retention window is sized to how long damage can go unnoticed, which for a record reviewed at renewal is months, rather than to how often the backup runs.
- A document-filing sweep runs every few minutes, moving each generated PDF into its protocol's folder and writing the link back to the record. It matches files to records by record key, not by protocol identifier, for the reason in the architecture section.
- A scheduled health monitor runs about two dozen checks over the data and the integrations, including a read-only heartbeat against the edge worker that distinguishes a missing secret from an unreachable service from an authentication drift. It arrives every day whether or not anything is wrong, so its absence is itself the alarm.
- A weekly report summarizes the week's movement for the coordinator and lists what needs attention, including expirations far enough ahead to act on. Expiration reporting is supersede-aware: a parent whose renewal has been approved stops generating noise, while a genuinely lapsed protocol still surfaces.
- An append-only audit log records every field-level change with actor and timestamp, supported by a snapshot table that lets a scheduled job diff state every fifteen minutes and detect changes made outside the app, including direct edits to the Sheet.

The audit log's honest limitations are documented in the file itself. Count-and-tail-hash stamping detects appends and truncation, not a mid-log edit; a chained-hash design is the fix, and it is deferred rather than pretended. And the actor it records is the last person to edit the row at diff time, so where a human action reliably triggers a bot write to the same row, the bot's account is recorded instead of the person. For the one field where that mattered, attribution moved into stamp columns written at the moment of the action, and the log tracks those columns as a tripwire: they are written only by automation, so any change to them in the log means someone edited an attribution by hand.

## Testing

Verification happened in several distinct modes, and they found different classes of defect.

**Adversarial role-switching**, run by the builder: run the entire lifecycle, intake through final status, as each role in turn, and try to do things each role should not be able to do. Two full end-to-end passes on clean data. This method, not unit-level checks, found every significant permission and automation defect: a security filter that silently blocked legitimate users, a status guard that silently swallowed a legitimate transition, a bot that missed flag-at-creation events, and a permission model where visibility had outrun mutation control. Every new feature since has been tested the same way, with explicit refusal cases alongside the positive one: the follow-up request, for example, was verified to appear where it should and to be absent on a draft, on a request with no response yet, on a resolved request, for the researcher, and for a different reviewer.

**Multi-person user testing**, run with real committee members and researchers on seeded data, each working from a role-specific packet with a written checklist, with the relay-critical steps sequenced so one person's output was the next person's input. In round one the full workflow relayed end to end across four people on the first attempt. Round two, with seven committee reviewers after an orientation, produced a short list of findings, all since built, including the follow-up request and a notice so that a withdrawn protocol no longer simply vanishes from a reviewer's queue. It raised nothing further.

The two modes are not interchangeable, and the difference is the most useful testing lesson from the project. **Testers exercising a system find discoverability problems**: a control they could not locate, a screen that hid the context they needed, a step whose order was unclear. **Domain experts looking at a system find correctness problems**: a workflow that does not match how the committee actually decides. Round one produced mostly the former from testers and the latter from the chair, which is an argument for giving domain experts review tasks rather than only use tasks.

**External reviews.** An accessibility review by the institution's disability resources office produced findings that split cleanly into what the system controls and what the platform controls. The first set was fixed; the second is recorded as platform limitation, with a published route for anyone who meets a barrier to get the task done another way. An information security review has been completed in conversation, with written confirmation pending.

Structural findings worth naming, because each was invisible to single-path testing:

- A completion check that examines only the record that triggered it misbehaves when a protocol carries several outstanding items of the same kind, and that only surfaces if the test data includes a protocol with two of them from two different people. Fixed, and retested with exactly that fixture.
- A generated export inherits none of the application's confidentiality rules. Resolved by the two-document design above.
- Test data needs the same shape as production data in every table, not just the one under test. Seeded records carried reviewer addresses in one form in the assignments table and another in the reviews table, which silently disabled the completion path described above. The visible symptom was a blank name in a reviewer column, the kind of thing that reads as cosmetic.
- A containment perimeter for test email has to cover every table that feeds a recipient list. Test addresses covered the researchers but not the reviewers, because reviewer recipients are derived from a different table, and the first test of a new notice reached two real reviewers directly. The perimeter was extended and re-verified on a live send.

One role is still untested by a second person, and the reason is structural rather than an oversight in planning. The builder occupies an operational role in the office, and a checklist the builder passes on their own role is not evidence: it leaves the largest role untested while appearing tested. That verification waits for a colleague. The veterinary and facility oversight paths have not yet been run by the people who will hold those roles, and that session is the last step before launch.

The dominant failure mode of low-code platforms is **silence**: misconfigurations do not error, they quietly do nothing. The countermeasures are procedural: a smoke-test ritual after structural changes, one clean diagnostic before any fix, and a maintenance reference that records not just every rule but the reasoning behind it, so the next maintainer can tell deliberate from accidental.

## Documentation as a verified artifact

The system is documented by a set of eleven documents: a user guide, role guides with screenshots, governance rules, a desk procedure, a build plan, a technical reference, a plain-language runbook organized by symptom, email mockups showing every notice, an ownership and succession document written to whoever inherits the system, a read-me-first overview, and an access and recovery inventory that names every account and service without holding any secret.

Documentation drifts from the system it describes, and it drifts silently for the same reason the platform fails silently: nothing checks. The practice that worked was to verify documents against the live system rather than against the previous version of themselves. A pass done that way found claims that had been wrong for months while being carried forward through several revisions: the technical reference said it was attached inside the application, and it never had been; a governance rule described a step as switched off while the office was using it; two role descriptions had fallen behind the permissions they described. None of those were typos. Each would have sent a successor to the wrong place.

## Results

One week from first login to: a functionally complete multi-role workflow covering the full protocol lifecycle including renewals and modifications, an automation chain, a self-healing data layer, a layered integrity model adopted by the office as policy, two full adversarial verifications, and a maintained document set. The week also included a deliberate data wipe and recovery, a root-caused regression, and several design reversals absorbed cleanly, the operational history most projects only accumulate after launch.

The months since added the parts that make a system survivable: a real intake pipeline, an operations layer that reports on itself, an audit trail, a confidentiality model with committee force behind it, two rounds of multi-person testing, an accessibility review, institutional ownership, and a documentation set covering every role.

The evolution is worth reading as a list, because most of it is the kind of work that only appears after a system meets its users:

- A coordinator intake step and a chair-review step were added at the front, so every protocol is prepared and classified before review begins, and a chair-approval step and a termination step at the end, each with its own stamped timestamps and record locking.
- A final coordinator check was added between review and approval, so training and medical clearance are confirmed before the chair sees a protocol.
- Three off-path statuses were added: paused, withdrawn, and new protocol required, along with chair outcomes on renewals and modifications.
- A view-only Oversight role was added for management, which prompted a full audit confirming that read access and mutation are governed independently.
- Veterinary and facility oversight became first-class roles with their own determination step and their own branch of the completion gate, and facility oversight later moved from role-based to assignment-based access.
- Intake moved from a form to a document-extraction pipeline, then gained a gate that will not let a protocol leave intake until the researcher can actually open the application.
- Revision requests gained follow-ups: linked records, resolved independently, shown as a chain in the interface and in the committee's record.
- The single generated PDF became two documents, one internal and one shareable.
- The identifier-is-not-a-key defect class was closed by a sweep rather than by fixing instances.
- The bot-invoked-identity defect class was closed the same way.
- The application, scripts, triggers, and alarms moved from a personal account to a departmental one.
- The identifier scheme gained a review-level suffix assigned by the chair at classification, a one-year expiration calculation was corrected from a fixed-day offset to an exact same-date-next-year computation so it stays accurate across leap years, and expiration reporting became supersede-aware.

Not everything is closed, and the open items are as instructive as the closed ones:

- The audit log cannot yet detect a mid-log edit.
- The single-role user model is resolved for facility oversight by moving to assignment-based access, but whether one person should be allowed to act as both reviewer and facility manager on the same protocol is a committee question, not yet answered.
- One integration still runs on a personal account.
- No live protocol has yet been processed, so real-world volume and edge cases are still ahead, and the extraction step's provider must be settled before live data reaches it.
- The oversight path awaits its session with the people who will hold those roles, and the coordinator path awaits a second person.

See [docs/lessons-learned.md](docs/lessons-learned.md) for the engineering lessons, which are the genuinely transferable output.

## Author

Built and documented by Spencer Steinberg. This case study is personal work product describing generalizable architecture and lessons; it intentionally contains no institutional information.
