# IACUC Protocol Review System

A case study: building a multi-role research compliance workflow system in Google AppSheet, from zero platform experience to a feature-complete, twice-verified system in one week.

This repository documents the architecture, security model, and engineering lessons. It contains no institutional data, no real protocols, and no operational configuration. Names, identifiers, and institution-specific details are omitted or genericized.

## The problem

A university research compliance office reviews animal-use protocols through an IACUC (Institutional Animal Care and Use Committee). The existing process ran on a legacy web platform, email threads, and human memory: reviewer assignments tracked by hand, revision rounds living in inboxes, statuses updated when someone remembered, and no enforced separation between what a researcher, a reviewer, and a committee chair should each see.

Constraints: one builder (a coordinator, not a developer), no budget, no development team, and the system had to run on tools the office already had, Google Workspace.

## The architecture

Three layers, deliberately boring:

- **Database:** a Google Sheet. One tab per table, stable UUID keys, references by ID. At a compliance office's scale (roughly a hundred protocols a year), a Sheet is a perfectly good database, and it gives you version history, trivial backup, and a data layer the office can read without the app.
- **Application:** Google AppSheet over the Sheet. Tables, role-based security filters, action conditions, format rules, and automation bots. AppSheet is the interface and rules layer; the Sheet stays the truth.
- **Automation:** Google Apps Script attached to the Sheet for the jobs AppSheet cannot do: normalizing form-submission intake into the protocol table, a nightly timestamped backup, and a self-healing routine that keeps a load-bearing derived column populated (more below).

Seven tables: Protocols, Users, Assignments, Reviews, RevisionRequests, PIResponses, CoordinatorNotes. Users drives everything: a Role column (Coordinator, Reviewer, PI, Chair, Oversight) that every security filter and action condition looks up by the signed-in email. Renewals and modifications are modeled without a new table: a protocol carries a submission type and an optional self-reference to its parent, so each renewal or modification is its own record with its own review lifecycle, linked to the original, which is never edited once approved.

## The role model

Five roles, very different systems:

| Role | Sees | Can do |
|---|---|---|
| Coordinator | Everything | Maintains all data, assigns reviewers, resolves revision rounds, keeps working notes, approves nothing but drives the lifecycle |
| Reviewer | Only assigned protocols, with co-reviewers' reviews and the researcher's responses | Writes and signs reviews; drafts and sends revision requests |
| Researcher (PI) | Only their own protocols; only revision requests actually sent to them | Submits immutable responses; does not edit the protocol record at all |
| Chair | Everything | Reviews and classifies each new protocol and assigns its reviewers at the front of the workflow, and approves the protocol at the end; alters nothing between those two points |
| Oversight | Everything, including coordinator working notes | Nothing; view-only across the entire system |

Visibility is enforced by security filters (rows a role cannot see are never loaded to their device, data-level privacy, not hidden UI). Mutation is enforced separately by action conditions and column-level editability. Keeping those two layers distinct was one of the central lessons: **visibility rules and mutation rules fail independently, and auditing one tells you nothing about the other.** Adding the view-only Oversight role made this concrete: granting it read access to every table exposed a set of note-management actions that had never carried role conditions, because they had relied on a security filter to stay hidden. View-only is not achieved by withholding views; it is achieved by withholding every mutating action, which must be verified action by action.

## The integrity model

The system's design center is that a compliance record must be trustworthy, which produced four rules:

1. **Signed reviews are permanent.** Signing is a deliberate, confirmed action; once signed, no role, including coordinators, can edit or delete the review through the app. The rare correction is a data-layer act, noted in the record.
2. **Researcher responses are immutable.** Written only by the researcher, unchangeable by anyone once submitted. A wrong response is corrected by a new response, preserving the original.
3. **Reviewed content is stable.** The researcher does not edit the protocol record; all edits are made by a coordinator (or by the chair during the front-end review step). Content does not shift under a reviewer mid-review, so signatures always refer to the text that was actually reviewed.
4. **Drafts are private and inert.** A revision request is an editable private draft until the reviewer deliberately sends it; an unsent draft never emails anyone, never changes a status, and is never visible to the researcher.

## The lifecycle and automation

A protocol now begins under chair review: every new record starts in a chair-review state where the chair classifies it (number, level, oversight flags) and assigns its reviewers, then advances it into review with a single confirmed action. From there, statuses are earned, not remembered:

- Sending a revision request advances a protocol to Revisions Requested.
- Signing a review, or resolving a revision request, triggers a completion check: when every review is signed and every sent request is resolved, the protocol advances itself to the chair-approval status.
- A revision request sent after completion automatically reopens the protocol.
- At the end, the chair approves the protocol with a confirmed action that stamps an approval timestamp and an expiration exactly one year ahead, then locks the record's status. Ending an approved protocol early is a separate, deliberate coordinator action that records a termination timestamp and freezes the record.

The completion and reopen transitions run as platform bots with transition-safe conditions; the chair-review advance, the approval, and the termination run as explicit role-gated actions rather than bots, because each represents a human decision that should be taken deliberately rather than triggered as a side effect. (An earlier design advanced a protocol automatically the moment a reviewer was assigned; once the chair-review step was added at the front, that automatic advance was retired, because assignment now happens while the chair is still working and must not move the protocol on its own.)

One derived column sits outside the bots and is worth calling out, because it is where the platform's silent-failure tendency bit hardest. Cross-row visibility (a reviewer seeing co-reviewers on a shared protocol) is driven by a real column that aggregates reviewer emails per protocol, computed by a Sheet formula rather than an in-app formula, because a security filter is evaluated at sync time and an in-app formula would go stale the instant a different table changed. A Sheet formula, though, can be silently erased by a data wipe or never written onto a script-created row. The fix is a self-healing script function: it walks the column, leaves correct cells alone, and rewrites the formula into any cell missing it, running on every intake and nightly. Building it surfaced a latent bug in the formula itself (a row-anchored range that would have silently dropped data), an instance of a safeguard auditing the thing it protects before it was ever needed.

Every bot uses transition-safe conditions (`AND([Flag] = TRUE, [_THISROW_BEFORE].[Flag] <> TRUE)`) and listens to both row creation and row updates, for reasons documented in the lessons file.

## Testing

The verification method was adversarial role-switching: run the entire lifecycle, intake through final status, as each role in turn, and try to do things each role should not be able to do. Two full end-to-end passes on clean data. This method, not unit-level checks, found every significant defect: a security filter that silently blocked legitimate users, a status guard that silently swallowed a legitimate transition, a bot that missed flag-at-creation events, and a permission model where visibility had outrun mutation control.

The dominant failure mode of low-code platforms is **silence**: misconfigurations do not error, they quietly do nothing. The countermeasures are procedural, a smoke-test ritual after structural changes, and a maintenance reference that records not just every rule but the reasoning behind it, so the next maintainer can tell deliberate from accidental.

## Results

One week from first login to: a functionally complete multi-role workflow covering the full protocol lifecycle including renewals and modifications, an automation chain, a self-healing data layer, a layered integrity model adopted by the office as policy, two full adversarial verifications, and a maintained document set (user guide, governance rules, build plan, technical reference). The week also included a deliberate data wipe and recovery, a root-caused regression, and several design reversals absorbed cleanly, the operational history most projects only accumulate after launch.

The system has continued to evolve since that first week. A chair-review step was added at the front of the workflow, so the committee chair classifies each protocol and assigns its reviewers before review begins; a chair-approval step and a termination step were added at the end, giving the record a complete lifecycle from intake through approval and, where needed, early termination, each with its own stamped timestamps and record-locking. A view-only Oversight role was added for management, which prompted a full audit confirming that read access and mutation are governed independently. A handful of now-unused early constructs were retired, and a one-year expiration calculation was corrected from a fixed-day offset to an exact same-date-next-year computation so it stays accurate across leap years.

See [docs/lessons-learned.md](docs/lessons-learned.md) for the engineering lessons, which are the genuinely transferable output.

## Author

Built and documented by Spencer Steinberg. This case study is personal work product describing generalizable architecture and lessons; it intentionally contains no institutional information.
