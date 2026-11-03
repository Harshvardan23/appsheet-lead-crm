# Enumerations and reference values

## Lead status (Online pipeline)

Column `Updated Status` on the processed-lead table. Type `Enum`, base type `Text`, "allow other values" ON. Display name **"Status After Call"**.

Presented in this order — the order is the pipeline, so it is not alphabetical and should not be re-sorted:

| # | Value | Stage |
|---|---|---|
| 1 | New Lead | Untouched |
| 2 | Not Answered | Contact attempted, no connection |
| 3 | Follow up | Connected, needs another call |
| 4 | Qualified | Meets criteria |
| 5 | Not Qualified | Does not meet criteria |
| 6 | In-transit | Moving toward placement |
| 7 | Offer Accepted | Offer accepted |
| 8 | Offer diff role | Offered a different role than applied for |
| 9 | Date of Joining Confirmed | Start date agreed |
| 10 | Joined | Placed — terminal success |
| 11 | Dropout | Withdrew after progressing — terminal |
| 12 | Not Interested | Declined — terminal |

Three terminal states, two of them negative. The distinction between **Not Interested** (declined early) and **Dropout** (withdrew after progressing) matters for reporting: the second represents work already spent.

> **The column name `Updated Status` must not change.** A lookup expression binds to it by string. The display name carries the user-facing wording.

## Call disposition

Reference table, 16 values. Separate from status by design: *status* is where the lead now stands, *disposition* is what happened on this particular call. A single lead can hold status "Follow up" across several calls with a different disposition on each.

## Rank

Numeric priority 0–5, assigned upstream. Key column `rank_value` (Number).

A blank rank is treated as Rank 0 everywhere — in the count, in the related-rows list and in the filtered drill-through:

```
OR([Rank] = [_THISROW].[rank_value],
   AND(ISBLANK([Rank]), [_THISROW].[rank_value] = 0))
```

Without the second clause, unranked leads vanish from the pipeline entirely — visible nowhere, counted nowhere. This was a real defect: leads existed in the Sheet and could not be reached from the app.

Label is generated rather than stored: `CONCATENATE("Rank ", [rank_value])`.

## Pipeline statuses (legacy)

Reference table of 13 stage names, used by the original status-based pipeline deck. Superseded by the rank pipeline; the table and its deck remain but are not on the navigation bar.

## Gender

`Enum`: Male / Female. Two values, as collected upstream.

## Willing to relocate

`Yes/No` with custom display labels rather than a two-value Enum — gives a toggle rather than a dropdown, which is faster on a phone.

## Qualification

Reference table rather than an Enum, so it can be extended by a manager in the app without an editor change. Referenced as `Ref → Qualifications`.

## Geography

Two reference tables: **India States** (28 states + 8 union territories) and **India Cities**.

Both are referenced twice from the lead record — current location and preferred location — so a candidate's willingness to move is a structured comparison rather than free text.

> India Cities is keyed on its row number rather than on the city name. This makes `REF_ROWS` against it fragile. Reference tables should be keyed on their value column.

## Currency

`Current Salary` and `Expected Salary` are type `Price`: 2 decimal places, thousands separator, INR symbol.

## Year of Passing

Type `Number` with thousands separator ON, so a graduation year renders as `2,024`. A known cosmetic defect — it should be an `Enum` of plausible years, or at minimum have the separator turned off.
