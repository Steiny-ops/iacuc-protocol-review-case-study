# Lessons Learned

Hard-won, specific, and transferable. Each of these cost real debugging time; together they are the most valuable output of the build.

## Security and permissions

**1. Security filters evaluate before virtual columns exist.**
AppSheet evaluates security filters at sync time, before virtual columns are computed. A filter that references a virtual column does not error; it silently fails to match. A feature was rebuilt three times before this root cause was identified. Rule: security filters may reference only real (sheet-backed) columns. If a filter needs derived data, materialize it into a real column first.

**2. A filtered-out row is invisible to LOOKUP too.**
Security filters do not just hide rows from views; they remove them from the user's entire data universe. A Users-table filter that excluded some users meant those users loaded zero Users rows, so every role lookup for them returned blank, and everything downstream of role checks silently broke. Directory-style tables that everything else depends on should usually load for everyone.

**3. Visibility and mutation are separate systems; audit both.**
Security filters govern what loads; action conditions and column editability govern what changes. Getting visibility right says nothing about mutation: a user who can legitimately *see* a record may inherit a default Edit on it. After any visibility work, separately ask: of everything each role can now see, what can they *change*?

**4. Hiding a field is not protecting it.**
A Show_If hides a column from the screen, but the data still syncs to the device. For genuinely private material (internal coordinator notes), use a separate table with its own security filter, so other roles never load the rows at all.

**5. Maintain one registry of people, not two.**
A hardcoded email whitelist inside a filter is a second user registry that will drift from the real one. Drive all access from the single Users table; adding a person should be one row in one place.

## Automation

**6. Bots need transition conditions, not state conditions.**
A bot conditioned on `[Flag] = TRUE` fires on every edit to a flagged row. Condition on the transition instead: `AND([Flag] = TRUE, [_THISROW_BEFORE].[Flag] <> TRUE)`. Fires exactly once, when the flag flips.

**7. Listen to Adds as well as Updates.**
A user who creates a row with the flag already set (writing and signing a review in a single form save) produces an Add, not an Update. An Updates-only bot misses it silently. With transition conditions, enabling Adds is safe, because a new row's before-value is blank.

**8. Adding a workflow status means auditing every status guard.**
Guards enumerate legitimate source states. Adding a new state silently changes what every existing guard means: a "set status" action guarded by one source state swallowed, without any error, a legitimate transition from the newly added state. When a status is added, list every status-setting action and re-derive each guard's source-state list.

**9. Platform bots fire only on in-app changes.**
Rows written directly into the backing Sheet, including rows created by scripts, do not trigger AppSheet bots. Intake pipelines must send their own notifications, and a hand-typed test row will never move a status. Test automation through the app, as a user.

## Data and platform behavior

**10. Blank is not FALSE.**
In AppSheet, a blank Yes/No column is not equal to FALSE. Conditions written as `[Signed] = FALSE` silently fail on rows where the cell is blank. Write `[Signed] <> TRUE`, which treats blank as unsigned, the safe direction. Generalize: for any nullable boolean, decide which way blank should fail, and write the comparison so it fails that way.

**11. Verify the key took.**
If no single column ends up marked as the key when adding a table, the platform silently manufactures a computed key (a concatenation) and displays it in views. Check the key explicitly after adding any table; fix by marking the intended column and deleting the synthetic one.

**12. Manual view column lists never pick up new columns.**
Once a view's column order is set to Manual, new columns and new related lists must be hand-added to every such view. A missing section usually is not broken; it is unlisted, rendering in a different position, or hidden by a role-based Show_If. Check which identity you are previewing as, and which surface you are looking at (a separately opened preview tab can pin an old app version), before debugging structure.

**13. Sheet-cell formulas are fragile infrastructure.**
A formula living in sheet cells (a TEXTJOIN aggregating per-row) can be silently destroyed by data wipes or row operations, and nothing complains. Prefer self-applying forms (ARRAYFORMULA) and document every sheet-side formula as infrastructure with a restoration procedure.

**14. Clear data with row deletion, not the Delete key.**
Clearing cell contents leaves ghost rows that scripts and appends treat as occupied, so new rows land a thousand rows down. Delete the rows themselves, and verify with Ctrl+End that the sheet's real extent is what you think it is.

## Process

**15. The dominant failure mode is silence.**
Across every defect found: nothing errored. Filters matched nothing, guards swallowed transitions, bots skipped events, all quietly. Defenses are procedural: an end-to-end smoke test as every role after structural changes, and documentation that records the reasoning behind each rule, so deliberate can be distinguished from accidental by the next maintainer.

**16. Test as the user, not as the builder.**
Builder-mode testing exercises happy paths with full permissions. Every significant defect was found by switching identities and running the whole lifecycle as each role, including trying to do what each role should not be able to do. Adversarial role-switching is the highest-yield testing hour available on a permissioned system.

**17. Documentation that describes reality is a feature.**
Documents were synced to the system at the end of every build day, with explicit tags separating what is working from what is planned. A maintenance reference that includes the failure modes and the must-not-retry list (approaches that were tried and abandoned, with the root cause) prevents the most expensive kind of rework: re-attempting a dead end.
