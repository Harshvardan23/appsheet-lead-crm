# Table: Call Log

**Source:** Google Sheet `<SHEET_ID>`, tab `Call Log` · **12 columns** · **Key:** `call_log_id`

One row per call attempt. Append-only in practice: nothing modifies a historical row except the status push, which only ever touches the most recent one.

## Columns

| Column | Type | Flags |
|---|---|---|
| call_log_id | Text | **K**, REQ, `i=UNIQUEID()` |
| lead_id | **Ref → Leads Raw** | |
| full_name | Text | **L** |
| phone_number | Text | |
| telecaller_email | Email | |
| call_start_time | DateTime | |
| call_end_time | DateTime | |
| call_duration_min | Number | |
| status_before_call | Text | |
| status_after_call | Text | |
| remarks | LongText | |

## Design notes

**`lead_id` is a `Ref`, not Text.** This is what makes the 58-field copy action possible: the action dereferences `[lead_id].[<column>]` to reach the raw lead's values. Without the ref, every field would need its own lookup.

**`telecaller_email` stores the email, not a display name.** Contrast with the WMS call log, which stores a resolved name and pays for it in every lookup. Store the email.

**`full_name` and `phone_number` are snapshots, not refs.** Denormalised deliberately: the open-calls view needs to show who is being called without resolving a ref, and a phone number that changes upstream should not retroactively change what a past call record says was dialled.

**An "open call" is a row with a start time and no end time.** No status flag, no boolean. The state cannot drift because it is not stored.

**Duration is computed on end**, never entered:
```
ROUND((TOTALSECONDS(NOW() - [call_start_time]) / 60.0) * 100) / 100.0
```
Two decimal places of minutes.

## Slice

**My Open Calls**
```
AND([telecaller_email] = USEREMAIL(), ISBLANK([call_end_time]))
```
All columns · slice action **End Call** · update mode: updates + adds + deletes

> The slice action has to be attached explicitly. It is not inherited from the table, and without it the open-calls view has no End Call button and the whole flow dead-ends.

## Actions on this table

| Action | Prominence |
|---|---|
| End Call (grouped) | **Prominent** |
| Set End Time | Hide |
| Ensure Processed Lead | Hide |
| Ask Status After Call | Hide |
| Set Status After Call | Hide |

Definitions in [`../expressions/actions.md`](../expressions/actions.md).

## Growth

One row per call attempt, with no archiving plan. At steady telecalling volume this table grows faster than anything else in the system, and several virtual columns elsewhere do full `SELECT` scans over it — the team rollups in particular.

No threshold has been hit, but the eventual fix is a periodic roll-off of closed calls into an archive tab, with the rollups reading a materialised summary instead of scanning. Noted rather than built.
