# Lessons learned

Rules derived from things that broke, in rough order of how much time they cost.

---

## 1. Slice views do not inherit table actions

A slice is a filtered view of a table. It does **not** inherit the table's custom actions. A view built on a slice shows none of them.

The failure is silent: the view renders, the data is correct, and the button simply is not there. There is no warning and no error.

**Rule:** every slice-backed view that needs a button needs that action added explicitly under the slice's own action list. Budget for adding each action twice.

## 2. Changing a column type must be done on every table that references it

Converting a column from `Ref` to `Text` on one table leaves every other table's reference to it pointing at the old type. Existing `REF_ROWS` virtual columns keep their definitions and quietly return nothing.

The app saves without error. The breakage appears at runtime as an empty list.

**Rule:** before changing a type, search every table for references to that column, and change them in the same sitting. Afterwards, re-check every virtual column that referenced it.

This left a live inconsistency in the system: on the processed-lead table `City` is `Text` while `State` is still a `Ref`, and the city reference table's `REF_ROWS` formula is stale as a result. Documented rather than fixed, because the fix touches the live pipeline.

## 3. Clear stale initial values when changing a type

A column converted from `Time` to `Text` kept its `TIMENOW()` initial value and started writing unparseable values into new rows.

**Rule:** changing a type is two steps. Change the type, then clear or rewrite the initial value.

## 4. `Require` on a target column breaks automated row-copies

An action that adds a row to another table will fail if that table has a required column the action does not set. The failure is silent — no row, no error, no log entry.

This was the cause of a day spent debugging a row-copy that "sometimes didn't work". It never worked; the cases that appeared to work were rows created by hand.

**Rule:** any table written by an automated action should have `Require` **off** on everything that action does not explicitly set. On the processed-lead table, that means `Remarks` and `Call Disposition` in particular.

## 5. Never rename a column referenced by an expression

AppSheet does not refactor. Renaming a column leaves every `LOOKUP`, `SELECT` and `REF_ROWS` referencing it by its old string name, and those are not flagged.

**Rule:** change the **display name**, never the column name. The display name is what users see; the column name is what expressions bind to.

A column in this system is called `Updated Status` and displays as "Status After Call", purely because a lookup expression references the former. It stays.

## 6. Read-only-to-humans is `Editable_If = FALSE`

The base `Editable?` checkbox makes a column read-only to everyone — including actions. `Editable_If = FALSE` makes it read-only to people while leaving it writable by automation.

**Rule:** for any column that a human should not touch but a copy action must write, use `Editable_If = FALSE`.

## 7. Guard every "start" action with a count

```
COUNT(SELECT(Call Log[call_log_id],
  AND(ISBLANK([call_end_time]),
      [lead_id] = [_THISROW].[Lead ID]))) = 0
```

Without it, a double-tap or a connection stutter produces two open rows and the duration data becomes unusable.

**Rule:** every action that creates a "pending" record carries a condition proving no pending record already exists. Same pattern for the row-copy, keyed on the target table's key.

## 8. Match Number against Text with `TEXT()` on both sides

A lookup from a table where the key is `Text` into a table where it is `Number` returns blank. There is no coercion and no error.

**Rule:** `TEXT([lead_id]) = TEXT([_THISROW].[lead_id])`, always, when the two sides might differ.

This bug is still live on the frozen WMS side: a phone-number lookup returns blank for exactly this reason.

## 9. Hiding a tab in Google Sheets does not hide it from AppSheet

AppSheet reads tabs regardless of their visibility in the Sheets UI. A tab hidden to keep it out of a collaborator's way is still offered as a table, still syncs, and still counts.

## 10. The bottom bar holds five views

Hard limit. Everything beyond five has to be reached by navigation from another view. This forces a decision about what the primary entry points actually are — which turned out to be useful, but it is better to make that decision deliberately than to discover the cap at the end.

## 11. The free-plan user cap is per app and counts monthly active users

Not per account. Not the user list. Anyone who opens the app in a month counts, including the developer, including a second account belonging to the same person, including someone who opened it once to look.

**Rule:** if a free-tier app is near the cap, do not share the install link to "just have a look". And use one account for yourself, not two.

This is what eventually forced the pipeline split in [06-rebuild.md](06-rebuild.md).

## 12. Deleting a table orphans its actions

Custom actions belonging to a deleted table are not deleted with it. They are re-parented to the user-settings table, where they are easy to miss. They have to be cleaned up separately, after the table is gone.

---

## The meta-lesson

Everything above has the same shape: **AppSheet fails silently far more often than it fails loudly.** The error panel catches structural problems — a missing table, a malformed expression — but almost nothing semantic. A broken lookup returns blank. A missing action renders an empty toolbar. A blocked row-copy does nothing at all.

The practical consequence is that **nothing can be assumed to work because it saved without error.** Every change needs a round trip through the actual app with real data. The test checklist in [06-rebuild.md](06-rebuild.md) exists because every item on it is something that once appeared to be configured correctly and was not.

The second consequence is that **deleting is safer than retyping.** When a platform gives you no diff, no version control and no test suite, a copy-and-trim migration is lower risk than a rebuild, even though a rebuild feels cleaner.
