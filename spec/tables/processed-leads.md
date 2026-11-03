# Table: Processed Leads

**Source:** Google Sheet `<SHEET_ID>`, tab `Processed Leads` · **67 columns** · **Key:** `Lead ID`

Created on first contact by the `Ensure Processed Lead` action, which copies 58 fields from Leads Raw. This is the table telecallers write to, and the one reporting reads.

## Critical constraints

**`Require` must be OFF on every column the copy action does not set.** In particular `Remarks` and `Call Disposition`. A required column the action does not supply makes the row-add fail with no error surfaced anywhere.

**`Updated Status` must not be renamed.** The `Set Status After Call` action binds to it by string in a `LOOKUP`. Change the display name instead; it already displays as "Status After Call".

## Columns unique to this table

| Column | Type | Flags |
|---|---|---|
| Updated Status | **Enum** | Display name "Status After Call". 12 values, see [`../enums.md`](../enums.md). Allow other values ON |
| Call Disposition | **Ref → Call Dispositions** | `Require` **OFF** |
| Remarks | Text | `Require` **OFF** |
| Status Before Call | Text | *noEdit*, `f=[Status]` |
| Change Fields | Text | Written by the sync |
| Sync Status | Text | *hidden* |
| Sync Time | Date | *hidden*, `i=TODAY()` |
| Sync Error | LongText | *hidden* — set to `""` by the copy action |
| `_last_synced_at` | DateTime | *hidden* |
| Status status | Text | A renamed duplicate header from the Sheet. See note below |

## Ref columns

| Column | Target |
|---|---|
| Rank | Ranks |
| State | India States |
| Preferred State | India States |
| Preferred City | India Cities |
| Qualification | Qualifications |
| Call Disposition | Call Dispositions |

> **Known inconsistency:** `City` is `Text` while `Preferred City` is a `Ref`, and `State` is a `Ref`. `City` was converted to `Text` and `State` was meant to follow, but did not. As a result the city reference table's `REF_ROWS` virtual column is stale and returns nothing. Make both `Text`, or both `Ref`.

## Columns copied from Leads Raw

The remaining 58 mirror Leads Raw one-for-one. Note that several lose their strictness in the copy — `Closure Due On`, `Timeline`, `Stage Updated At` and `WhatsApp Opted In` arrive as `Text` here rather than their typed equivalents, because the copy carries the upstream string.

That is not ideal, but it is deliberate: a type conversion inside the copy action would fail on any malformed upstream value and take the whole row-add with it. Storing the raw string and converting at read time is more forgiving.

## Type and formatting settings that matter

| Column | Setting |
|---|---|
| Gender | Enum: Male / Female |
| Current Salary, Expected Salary | Price, 2 decimals, thousands separator, INR symbol |
| Willing to Relocate | Yes/No with custom display labels |
| Year of Passing | Number — **thousands separator should be OFF**, currently renders as `2,024` |
| Time Slot | Text. Was `Time` with a stale `TIMENOW()` initial value; both fixed together |
| Lead Source | `Editable_If = FALSE` — read-only to users, still writable by the copy action |
| Campaign Name | Name. Should be `Editable_If = FALSE`; currently editable |

## The "Status status" column

The Sheet originally carried two columns both headed `Status`. One was renamed to `Status status` to disambiguate. The sync script may still expect the original header — this needs confirming with whoever owns the sync before the sheet is touched again.

Named here exactly as it appears so a rebuild does not silently drop it.

## Build order for this table

1. Create the table from its Sheet tab and set `Lead ID` as key
2. Turn `Require` **off** on everything — add it back only where genuinely needed
3. Create the reference tables first, then set the six `Ref` columns
4. Create `Updated Status` as an Enum and add the 12 values in pipeline order
5. Set the display name on `Updated Status` to "Status After Call"
6. Add the `Status Before Call` virtual column
7. Set `Editable_If = FALSE` on `Lead Source` and `Campaign Name`
8. Build the copy action last — it references every column on this table
