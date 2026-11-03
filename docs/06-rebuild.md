# Splitting the live pipeline into its own app

## Why

AppSheet's free tier caps **prototype apps at 10 monthly active users, per app**. The main app crossed that line and was blocked: the editor still opened, but the app would not load for anyone.

Three things about that limit that were not obvious until they bit:

1. **It is counted per app, not per account.** Other apps on the same account showed zero users and were unaffected.
2. **It counts monthly active users, not the user list.** Anyone who opens the app counts, whether or not they appear in the sharing dialog. The user list showed a handful of people; the usage count was substantially higher.
3. **Multiple accounts belonging to the same person count separately.** Testing with a second Google account inflates the number.

The app had accreted three pipelines, two of which were retired or frozen, and was carrying their users along with the live one. The fix was to split the live pipeline into a smaller app with a tighter user list.

## Approach: copy and trim, not rebuild

Two options were considered.

**Build from scratch.** Full control, no inherited cruft. But the row-copy action alone sets 60 columns by hand, and every ref, guard and formula has to be retyped. Several hours, with a transcription error on any of them producing a silent runtime failure.

**Copy the app, then delete what is not needed.** The expressions, refs, guards, slice and form all arrive exactly as they are. Trimming is deletion, and a deletion that breaks something fails loudly with a named reference.

Copy-and-trim was chosen. The judgement: on a declarative platform with no diff and no test suite, *deleting* is far safer than *retyping*.

## What was kept

Eight tables:

```
Leads Raw · Processed Leads · Call Log
Ranks · Call Dispositions · Qualifications
India States · India Cities
```

Plus: all the call-flow actions, the open-calls slice, the rank pipeline deck, the lead and detail views, the End Call Status form, and the `[Account Owner Email] = USEREMAIL()` security filter.

## What was removed

| Category | Removed |
|---|---|
| Tables | All WMS tables, all Meta tables, the in-app README, the legacy status table |
| Slices | 14 of 15 — only the open-calls slice retained |
| Views | All WMS, Meta, manager and report views |
| Actions | 18 orphaned WMS/Meta actions |
| Bots | Both (neither did anything — see below) |
| Format rules | Both (both WMS) |

## Trim order

Order matters. AppSheet refuses to delete an object another object references, so work outward from the leaves:

1. **Views** first — they reference slices and actions
2. **Slices** — they reference tables and actions
3. **Actions** — they reference tables
4. **Format rules** and **bots**
5. **Tables** last

Check the error panel after each group. A deletion that breaks a reference names the referencing object in the error, which tells you exactly what to remove next.

### Two traps found during the trim

**Deleting a table orphans its custom actions rather than deleting them.** The actions do not disappear — they are re-parented to the user-settings table, where they sit invisibly. They have to be found and deleted there, after the table is gone.

**Slice and view deletions have no confirmation dialog.** Table deletions do. It is easy to delete a slice with one click and not notice.

## Post-trim configuration

- Starting view changed from a WMS detail view to the rank pipeline. The starting view is set in the UX options and does not update itself when its target is deleted.
- Bottom bar reduced to the two Online views.
- Sign-in settings re-verified: require sign-in ON, allow-all OFF, user list rebuilt from scratch rather than inherited.
- Security filter confirmed present on the raw leads table — it survives the copy, but is worth checking.

## Test checklist

Run with one real telecaller account and one real lead before letting anyone else in.

1. Sign in as a telecaller — **only** leads with that account owner email are visible.
2. Start Call & Dial — a call log row appears with start time and status-before-call; the dialler opens.
3. Start Call again on the same lead while the first is open — **blocked**.
4. End Call — end time and duration set; a processed-lead row exists **exactly once**; the status form opens.
5. Save the form with a status — the call log row shows that status.
6. Rank pipeline counts match the Sheet, including blank ranks appearing under Rank 0.
7. Error panel — no new errors.
8. Repeat 2–5 on a lead that already has a processed row — no second row is created.

## Open items

- **The copied app inherited the plan block.** Preview showed the same upgrade-required message as the original, which suggests the restriction is account-scoped or is inherited by a copy. Unresolved; needs a plan decision or a support query.
- **The upstream feed was not re-pointed.** The copy uses a duplicated Sheet; whatever writes the raw lead table — an import formula needing re-authorisation, or a sync script needing a new target — has to be redirected before the app has data.
- **The user list was not populated** beyond the owner.
- **The test checklist above has not been run**, because the app will not load.

## Two bots that did nothing

The original app had two automation bots. One was enabled, with a trigger wired to a process containing a single empty step. The other was disabled. Neither touched the live pipeline, and neither had ever done anything.

They were deleted rather than carried forward. Worth stating plainly: the automation layer of this system is zero. Everything happens in actions, triggered by a user tapping a button. For a workflow where every step follows a human picking up a phone, that is the correct design, not a gap — but it is a deliberate position rather than an oversight, and it means there is no retry, no queue and no scheduled reconciliation anywhere in the system.
