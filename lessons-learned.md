# Lessons Learned

Hard-won, specific, and transferable. Each of these cost real debugging time; together they are the most valuable output of the build.

## Security and permissions

1. **Security filters evaluate before virtual columns exist.** AppSheet evaluates security filters at sync time, before virtual columns are computed. A filter that references a virtual column does not error; it silently fails to match. A feature was rebuilt three times before this root cause was identified. Rule: security filters may reference only real (sheet-backed) columns. If a filter needs derived data, materialize it into a real column first.

2. **A filtered-out row is invisible to LOOKUP too.** Security filters do not just hide rows from views; they remove them from the user's entire data universe. A Users-table filter that excluded some users meant those users loaded zero Users rows, so every role lookup for them returned blank, and everything downstream of role checks silently broke. Directory-style tables that everything else depends on should usually load for everyone.

3. **Visibility and mutation are separate systems; audit both.** Security filters govern what loads; action conditions and column editability govern what changes. Getting visibility right says nothing about mutation: a user who can legitimately see a record may inherit a default Edit on it. After any visibility work, separately ask: of everything each role can now see, what can they change?

4. **Hiding a field is not protecting it.** A Show_If hides a column from the screen, but the data still syncs to the device. For genuinely restricted material, use a separate table with its own security filter, so roles that should not have the data never load the rows at all. (The same structure makes the access decision easy to revisit later: widening or narrowing who sees that table is a change to one filter's role list, not a rework, though any such change also means re-checking the table's own add, edit, and delete actions, since those are governed separately from the filter.)

5. **Maintain one registry of people, not two.** A hardcoded email whitelist inside a filter is a second user registry that will drift from the real one. Drive all access from the single Users table; adding a person should be one row in one place. The platform's own access list is a third registry, and it is the one that will lock legitimate users out while every in-app check passes, because it is enforced before the app loads at all.

6. **Identity-based security breaks on aliases.** A filter comparing a stored address to the signed-in user is only correct if the stored address is the account's primary address. A user whose record carries a valid alias authenticates successfully and then sees an empty list, with nothing anywhere reporting a problem. Anywhere identity drives visibility, require the canonical account address at data entry, and treat an empty list for an authenticated user as an identity mismatch until proven otherwise.

7. **Scheduled automations run without a user identity.** A bot that runs on a schedule has no signed-in user, so any function returning the current user returns blank, and every security filter that depends on it matches nothing. The table reads as empty and the job completes successfully having done nothing. Scheduled jobs that read filtered tables must be configured to bypass security filters, which is a deliberate privilege grant and should be recorded as one.

8. **A single-value enum and a multi-value enum are not interchangeable.** Every condition written as an equality comparison against a role column silently returns false if that column is later widened to hold multiple values. Nothing errors; dropdowns come back empty and conditions stop matching. Widening a role model is therefore not a schema change, it is a sweep of every security filter, every visibility condition, and every validity rule that names a role. Count those references before estimating the change.

9. **Constraining selection is not constraining creation.** A validity rule that filters a reference field to accounts holding a particular role governs which existing record can be chosen. If the reference field also offers inline creation, a user can create a new record from inside the picker, bypassing the constraint entirely and producing a record with no role and no working address. Restriction at assignment time and restriction at creation time are separate; check both.

## Automation

10. **Bots need transition conditions, not state conditions.** A bot conditioned on `[Flag] = TRUE` fires on every edit to a flagged row. Condition on the transition instead: `AND([Flag] = TRUE, [_THISROW_BEFORE].[Flag] <> TRUE)`. Fires exactly once, when the flag flips.

11. **Listen to Adds as well as Updates.** A user who creates a row with the flag already set (writing and signing a review in a single form save) produces an Add, not an Update. An Updates-only bot misses it silently. With transition conditions, enabling Adds is safe, because a new row's before-value is blank.

12. **Adding a workflow status means auditing every status guard.** Guards enumerate legitimate source states. Adding a new state silently changes what every existing guard means: a "set status" action guarded by one source state swallowed, without any error, a legitimate transition from the newly added state. When a status is added, list every status-setting action and re-derive each guard's source-state list.

13. **Platform bots fire only on in-app changes.** Rows written directly into the backing Sheet, including rows created by scripts, do not trigger AppSheet bots. Intake pipelines must send their own notifications, and a hand-typed test row will never move a status. Test automation through the app, as a user.

14. **A completion check must count siblings, not inspect its trigger.** A bot that fires when an item is resolved and then evaluates only the item that fired it will report completion while identical items are still outstanding on the same parent. This is invisible in any test where a record carries one of a thing, and it appears the moment a record carries two. Write completion checks as a count over the parent's children, and build at least one fixture that carries two of everything countable.

15. **Test scheduled jobs with a real run, not the test panel.** Test harnesses commonly mask personally identifying fields and supply synthetic context, so a job can pass in the panel and fail in production for reasons the panel deliberately hid. Trigger a real run and read the execution log.

16. **Per-task previews render local columns only.** A template preview that resolves fields on the current record will show correct output for those fields and blank for anything reached through a reference, even when the real run resolves everything. Preview blanks are not evidence of a broken template.

## Data and platform behavior

17. **Blank is not FALSE.** In AppSheet, a blank Yes/No column is not equal to FALSE. Conditions written as `[Signed] = FALSE` silently fail on rows where the cell is blank. Write `[Signed] <> TRUE`, which treats blank as unsigned, the safe direction. Generalize: for any nullable boolean, decide which way blank should fail, and write the comparison so it fails that way.

18. **Verify the key took.** If no single column ends up marked as the key when adding a table, the platform silently manufactures a computed key (a concatenation) and displays it in views. Check the key explicitly after adding any table; fix by marking the intended column and deleting the synthetic one.

19. **Format every identifier column as plain text.** An identifier that happens to be all digits will be coerced to a number by the spreadsheet, which strips leading zeros and breaks every reference pointing at it. This applies to every identifier column in every table, not only the obvious primary keys, and it must be reapplied after any operation that recreates a sheet.

20. **Manual view column lists never pick up new columns.** Once a view's column order is set to Manual, new columns and new related lists must be hand-added to every such view. A missing section usually is not broken; it is unlisted, rendering in a different position, or hidden by a role-based Show_If. Check which identity you are previewing as, and which surface you are looking at (a separately opened preview tab can pin an old app version), before debugging structure.

21. **Some fields are expressions even when they look like text.** Recipient and display-name fields in message templates are evaluated as expressions, so a literal value entered without quoting is parsed as a formula and throws a type error, or worse, concatenates with an adjacent value and produces something plausible. If a field accepts a formula, assume it evaluates one, and quote literals.

22. **A filter-backed derived column must be computed in the sheet, not in-app, and kept alive by a self-healing script.** A column a security filter depends on cannot be an in-app formula: those recompute only when the row itself is saved, so a value derived from another table goes stale the instant that other table changes. It has to be a sheet formula, which recomputes on any sheet edit. But a sheet formula can be silently destroyed by a wipe or never written onto a script-created row, and nothing complains. The durable fix is a small script function that re-writes the formula into any row missing it, run on every intake and nightly, with a stored canonical copy for the case where no healthy cell remains to copy from. Building that safeguard is also what caught a latent row-anchoring bug in the formula, the safeguard audited the thing it protected.

23. **Convert a formula to R1C1 before a script copies it across rows.** A formula written with a row-anchored range (a fixed start row) becomes, when copied row-relative, a range that starts at each row's own position, silently dropping any data that sits above it. Whole-column references carry no row anchor and copy safely. Inspecting the formula in R1C1 form makes the anchoring visible before it can cause a silent miss.

24. **Clear data with row deletion, not the Delete key.** Clearing cell contents leaves ghost rows that scripts and appends treat as occupied, so new rows land a thousand rows down. Delete the rows themselves, and verify with Ctrl+End that the sheet's real extent is what you think it is.

## Monitoring and audit

25. **A validity check is not a reachability check.** A monitor that verifies a stored link is well formed reports green on a well-formed link to a destination that does not exist. The two checks answer different questions, and the shape check is the one that is easy to write. Where a destination matters, test that the destination resolves, and accept that this makes the check slower and occasionally noisy.

26. **Deduplication without a time dimension deletes real history.** A cleanup routine that treats records with identical field values as duplicates will delete legitimate repeat events, for example the same status transition occurring twice in a normal cycle. In an audit context this is worse than a no-op, because the routine then re-stamps its own integrity markers over deletions it just made, producing a log that certifies itself as intact. Any deduplication over an event log must key on time as well as content, and anything touching an audit trail should be assumed harmful until proven otherwise.

27. **Know what your tamper-evidence actually detects.** Count-and-tail-hash stamping over an append-only log detects appended and truncated records. It does not detect an edit in the middle of the log, because neither the count nor the tail changes. That is an acceptable limitation only if it is written down where the next maintainer will find it, next to the design that would fix it.

28. **Monitor the backup, not just the backup job.** A scheduled backup that completes successfully proves the job ran, not that anything usable was written. Destinations resolved by name rather than by identifier are the specific hazard: two destinations can share a name, and writes will land in whichever one the lookup returns, silently, while every log reports success. Pin destinations by identifier and add a freshness check on the artifact itself.

## Process

29. **The dominant failure mode is silence.** Across every defect found: nothing errored. Filters matched nothing, guards swallowed transitions, bots skipped events, all quietly. Defenses are procedural: an end-to-end smoke test as every role after structural changes, and documentation that records the reasoning behind each rule, so deliberate can be distinguished from accidental by the next maintainer.

30. **Test as the user, not as the builder.** Builder-mode testing exercises happy paths with full permissions. Every significant defect in the first phase was found by switching identities and running the whole lifecycle as each role, including trying to do what each role should not be able to do. Adversarial role-switching is the highest-yield testing hour available on a permissioned system.

31. **Testers find discoverability; domain experts find correctness.** These are different defect classes and they need different tasks. People given a checklist and a system will tell you what they could not find, which control confused them, and where the interface hid the context they needed. People who know the domain, looking at the same system, will tell you that the workflow does not match how decisions are actually made. If you want the second kind, give domain experts something to review rather than something to use.

32. **The builder cannot test their own role.** The person who built the system will pass every step of their own checklist, and a self-passed checklist leaves the largest role untested while appearing tested. If the builder occupies an operational role, that role's verification has to wait for someone else, and the gap should be recorded rather than papered over.

33. **Generated exports inherit none of the application's rules.** An interface can enforce a confidentiality model perfectly and then a generated document assembled from the same data can carry exactly what the interface withholds. Export templates are a separate surface with separate rules, and they are the surface most likely to leave the institution. Audit every generated artifact against the same access model the interface implements, and audit it again whenever the model changes. Discovering this late is common, because the export usually predates the confidentiality rule that it violates.

34. **Documentation drifts toward aspiration.** Documents written during a build describe the system as it is intended to be, and they keep saying that after reality moves. Several documents in this project asserted the system was not yet deployed weeks after it was serving live submissions. Sync documentation to reality on a schedule, tag what is working separately from what is planned, and treat any document that cannot be verified against the running system as a claim rather than a record.

35. **Documentation that describes reality is a feature.** Documents were synced to the system at the end of every build day, with explicit tags separating what is working from what is planned. A maintenance reference that includes the failure modes and the must-not-retry list (approaches that were tried and abandoned, with the root cause) prevents the most expensive kind of rework: re-attempting a dead end.

36. **Write down what you decided not to do, and why.** A deferred fix with a recorded root cause is an asset; a deferred fix remembered as "that was weird" is a bug that will be rediscovered from scratch. The same applies to constraints accepted deliberately: a limitation someone chose looks identical to an oversight six months later unless the reasoning is written next to it.
