# Data model

All tables share one Google Sheet workbook (`<SHEET_ID>`). The AppSheet table qualifier is the tab name.

Notation used throughout: **K** key · **L** label · **REQ** required · *noShow* hidden · *noEdit* not editable · `i=` initial value · `f=` formula (virtual or app formula).

---

## Online pipeline (live)

### Leads Raw — 61 columns

Mirror of the upstream WhatsApp CRM feed. Written by the sync, read by the app. Carries a row-level security filter.

| Column | Type | Notes |
|---|---|---|
| Lead ID | Text | **K**, REQ |
| Phone Number | **Phone** | REQ. Type must stay `Phone` — the dialler action depends on it |
| Name | Name | **L** |
| Email | Email | REQ |
| Account Owner Email | Email | REQ. Drives the security filter |
| Status | Text | Current stage from upstream |
| Rank | Ref → Ranks | REQ. Priority 0–5 |
| Campaign Name, Campaign ID, Course Name | Name / Text | Source attribution |
| State, City, Preferred State, Preferred City | Text | |
| Qualification, Year of Passing, Years of Experience | Text / Number | Candidate profile |
| Current Salary, Expected Salary | Price | INR, 2 decimals |
| Willing to Relocate | Yes/No | REQ |
| Lead Age (Days), Days to Closure | Number | Computed upstream |
| `_row_hash` | Text | Sync change detection |

Plus ~35 further profile, source-attribution and sync-bookkeeping columns. One virtual column: `Related Call Logs = REF_ROWS("Call Log", "lead_id")`.

**Security filter:** `[Account Owner Email] = USEREMAIL()`

### Processed Leads — 67 columns

Created on first contact by the row-copy action. This is where telecallers write.

Mirrors Leads Raw field-for-field, plus:

| Column | Type | Notes |
|---|---|---|
| Updated Status | **Enum** | Display name "Status After Call". 12 values — see [`spec/enums.md`](../spec/enums.md). **Do not rename this column**; a lookup expression references it by name |
| Call Disposition | Ref → Call Dispositions | `Require` must be **OFF** |
| Remarks | Text | `Require` must be **OFF** |
| Status Before Call | Text | *noEdit*, `f=[Status]` |
| Change Fields, Sync Status, Sync Time, Sync Error | Text / Date / LongText | Written by the sync |
| `_last_synced_at` | DateTime | |

Ref columns: Rank → Ranks · State, Preferred State → India States · Preferred City → India Cities · Qualification → Qualifications · Call Disposition → Call Dispositions.

> **Known inconsistency:** `City` is `Text` while `Preferred City` is a `Ref`, and `State` is a `Ref` while `City` is not. Documented rather than silently fixed — see [lessons-learned.md](lessons-learned.md).

### Call Log — 12 columns

One row per call attempt.

| Column | Type | Notes |
|---|---|---|
| call_log_id | Text | **K**, REQ, `i=UNIQUEID()` |
| lead_id | Ref → Leads Raw | |
| full_name | Text | **L** |
| phone_number | Text | |
| telecaller_email | Email | `USEREMAIL()` at call start |
| call_start_time, call_end_time | DateTime | |
| call_duration_min | Number | Computed on end |
| status_before_call, status_after_call | Text | |
| remarks | LongText | |

### Ranks

Priority buckets with live counts. Key `rank_value` (Number). Blank rank on a lead counts as Rank 0.

```
Rank Label = CONCATENATE("Rank ", [rank_value])

Lead Count = COUNT(SELECT(Leads Raw[Lead ID],
  OR([Rank] = [_THISROW].[rank_value],
     AND(ISBLANK([Rank]), [_THISROW].[rank_value] = 0))))
```

### Reference tables

Read-only lookups. Key and label are the single value column.

| Table | Key column | Contents |
|---|---|---|
| Call Dispositions | disposition_value | 16 call outcome values |
| Qualifications | qualification_value | Education levels |
| Statuses | status_name | 13 pipeline stages |
| India States | state_name | 28 states + 8 union territories |
| India Cities | City_name | City list |

> India Cities is keyed on `_RowNumber` rather than `City_name` — a legacy mistake. `REF_ROWS` against it is unreliable once a city column is converted to Text. Key reference tables on the value column.

---

## WMS pipeline (locked)

### Leads — 37 columns

| Column | Type | Notes |
|---|---|---|
| lead_id | Number | **K**, REQ |
| first_name, last_name, father_name, mother_name | Name | |
| mobile_number | Phone | *noEdit*, REQ |
| email_id | Email | REQ |
| lead_status | Enum | Pipeline stage |
| wms_user_name | Name | *noEdit*. Assigned telecaller; drives per-user slices |
| job_event_id, job_event_name, job_event_location | Number / Name / Enum | Recruitment drive |
| cost_to_company | Price | REQ |
| regional_head, mis_executive, lead_supervisor | Text | |
| `_ComputedName` | Name | **L**, `f=CONCATENATE([first_name]," ",[last_name])` |

### Call Log — 15 columns

As the Online call log, plus:

```
Status_changed = AND(NOT(ISBLANK([status_after_call])),
                     [status_after_call] <> [status_before_call])

call_date      = DATE([call_start_time])
```

> `lead_phone = LOOKUP([lead_id], "Leads", "lead_id", "mobile_number")` returns blank. `lead_id` is `Number` on Leads and `Text` on Call Log, and the lookup does not coerce. Fix is to `TEXT()`-wrap both sides. Left documented, not patched, because this side is frozen.

### Managers / Telecallers / Team Mapping

The role graph. A manager's team resolves as:

```
Managers (by Manager Email = USEREMAIL())
   └─► Team Mapping (by manager_id)
         └─► Telecallers (by telecaller_email)
               └─► Leads / Call Log (by wms_user_name)
```

Team Mapping also carries the per-telecaller rollups (`total_calls`, `total_duration`, `joined_count`), each bounded by the user-settable report date range.

### Assignment Quotas

| Column | Type |
|---|---|
| manager_name | Name, **K**, **L**, REQ |
| telecaller_email | Email |
| quota | Number, `i=0` |
| range_start, range_end | Number |

Position-band assignment: a lead at queue position *n* goes to whichever telecaller's `range_start`–`range_end` band contains *n*.

---

## Meta pipeline (retired)

### Leads Raw — 28 columns

Raw ad-platform payload: `id`, `created_time`, ad / adset / campaign / form identifiers and names, `is_organic`, `platform`, plus lead fields. All *noEdit*.

Four virtual columns do the cleaning:

```
meta_lead_key   = CONCATENATE([id], "|", [phone_number], "|", [created_time])
phone_clean     = RIGHT(SUBSTITUTE(SUBSTITUTE([phone_number],"p:",""),"+",""), 10)
is_test_row     = OR(CONTAINS(UPPER([campaign_name]),"TEST"),
                     CONTAINS(UPPER([campaign_name]),"SAMPLE"),
                     CONTAINS(UPPER([form_name]),"TEST"),
                     CONTAINS(UPPER([form_name]),"SAMPLE"),
                     CONTAINS(UPPER([full_name]),"TEST"))
already_assigned = IN([meta_lead_key], Leads[meta_lead_key])
```

### Leads Working — 31 columns

Same raw columns plus the assignment layer. Full formulas in [`spec/expressions/virtual-columns.md`](../spec/expressions/virtual-columns.md).

### Campaign Manager Mapping

Two columns: `campaign_name` (**K**, **L**) → `manager_name`. The entire ad-routing configuration — editable by a manager in the app, no redeploy.

---

## Column type rules learned the hard way

1. A column used by the dialler action must be type `Phone`. `Text` silently disables the button.
2. Changing a type on one table does not change it on tables that reference it. Change all of them in the same sitting.
3. Clear the initial-value formula when changing a type — a stale `TIMENOW()` on a column converted to Text produces unparseable values.
4. For read-only-to-humans-but-writable-by-automation, use `Editable_If = FALSE`, not the base Editable checkbox.
5. `Require` on a target column blocks an automated row-copy into that table, with no error surfaced to the user.
6. Matching a Number key against a Text key needs `TEXT()` on both sides.
