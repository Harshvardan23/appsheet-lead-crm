# Table: Leads Raw

**Source:** Google Sheet `<SHEET_ID>`, tab `Leads Raw` · **61 columns** · **Key:** `Lead ID`

Mirror of the upstream WhatsApp CRM feed. Written by the sync process, never by a user. Carries the app's only row-level security filter.

## Security filter

```
[Account Owner Email] = USEREMAIL()
```

Applied in Security → Security Filters. Evaluated server-side: rows a user does not own are never downloaded.

## Columns

### Identity and contact

| Column | Type | Flags |
|---|---|---|
| Lead ID | Text | **K**, REQ, *hidden* |
| Name | Name | **L** |
| Phone Number | **Phone** | REQ, *hidden* |
| Country Code | Number | REQ, *hidden* |
| Email | Email | REQ, *hidden* |
| User ID | Text | REQ, *hidden* |
| WhatsApp User ID | Text | |
| WhatsApp Opted In | Yes/No | REQ, *hidden* |

> `Phone Number` **must** be type `Phone`. The dialler action binds to it and is silently unavailable on a `Text` column.

### Ownership and stage

| Column | Type | Flags |
|---|---|---|
| Account Owner Email | Email | REQ, *hidden* — drives the security filter |
| Account Owner Name | Name | *hidden* |
| Status | Text | |
| Rank | **Ref → Ranks** | REQ, *hidden* |
| Owner UUID, Stage UUID | Text | *hidden* |
| Stage Updated At | Date | REQ, *hidden*, `i=TODAY()` |

### Source attribution

| Column | Type | Flags |
|---|---|---|
| Lead Source | Text | *hidden* |
| Internal Lead Source | Enum | *hidden* |
| Campaign Name | Name | |
| Campaign ID | Text | REQ, *hidden* |
| Course Name | Name | *hidden* |
| Entry Channel, Channel Type | Text | *hidden* |
| Source ID | Text | *hidden* |
| Source URL | Url | *hidden* |
| Tags | Text | *hidden* |
| Marked as Spam | Text | *hidden* |

### Candidate profile

| Column | Type | Flags |
|---|---|---|
| Gender | Text | |
| Age in Years | Number | REQ |
| Date of Birth | Date | REQ, `i=TODAY()` |
| Father's Name, Mother's Name | Name | |
| Qualification | Text | |
| Year of Passing | Number | REQ |
| Years of Experience | Number | REQ |
| Jobs | Number | REQ, *hidden* |
| Work Specialisation | Text | *hidden* |
| Company Name | Name | |
| Current Salary, Expected Salary | Price | INR, 2 dp |

### Location

| Column | Type | Flags |
|---|---|---|
| State, City | Text | |
| Preferred State, Preferred City | Text | |
| Current Pincode | Text | REQ |
| Location | Text | |
| Willing to Relocate | Yes/No | REQ |

### Pipeline and bookkeeping

| Column | Type | Flags |
|---|---|---|
| Call Disposition | Text | |
| Remarks | Text | |
| Creation Date, Last Modified | Date | REQ, *hidden*, `i=TODAY()` |
| Timeline | Date | REQ, *hidden*, `i=TODAY()` |
| Time Slot | Text | *hidden* |
| Closure Due On | Date | REQ, *hidden*, `i=TODAY()` |
| Deal Value | Price | *hidden* |
| Date of Joining | Date | *hidden*, `i=TODAY()` |
| Employee ID | Text | *hidden* |
| Lead Age (Days), Days to Closure | Number | REQ, *hidden* |
| `_row_hash` | Text | *hidden* — sync change detection |

### Virtual

```
Related Call Logs = REF_ROWS("Call Log", "lead_id")
```

## Notes

- Most columns are hidden. The raw table is not a working surface — telecallers see a short summary and tap through to the detail view.
- `Require` flags here are inherited from the upstream schema and do not block anything, because the sync always supplies values. Do **not** copy this pattern onto the processed table, where `Require` breaks the automated row-copy.
- Not built, but designed: `Last Called On` and `Latest Call Outcome` virtual columns resolving the most recent call via `MAXROW` against the call log. Adding them means two full scans of the call log per lead row, which is why they were deferred.
