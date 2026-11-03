# Actions

Exact definitions. Table names are generic; substitute your own.

Prominence matters: only the two grouped actions are visible to users. Everything else is set to **Hide** and runs only as a member of a group.

---

## Online pipeline

### Start Call  — *hidden*

**Table:** Leads Raw · **Type:** add a new row to another table using values from this row · **Target:** Call Log

**Condition:**
```
COUNT(SELECT(Call Log[call_log_id],
  AND(ISBLANK([call_end_time]),
      [lead_id] = [_THISROW].[Lead ID]))) = 0
```

**Sets:**
```
call_log_id        = UNIQUEID()
lead_id            = [Lead ID]
full_name          = [Name]
phone_number       = [Phone Number]
telecaller_email   = USEREMAIL()
call_start_time    = NOW()
status_before_call = [Status]
```

### Start Call & Dial  — *prominent*

**Table:** Leads Raw · **Type:** grouped

1. Start Call
2. Call Phone (system action, on `[Phone Number]`)

Display name: "Start Call & Dial". The phone column must be type `Phone` or the second step is unavailable.

### Set End Time  — *hidden*

**Table:** Call Log · **Type:** set column values

```
call_end_time     = NOW()
call_duration_min = ROUND((TOTALSECONDS(NOW() - [call_start_time]) / 60.0) * 100) / 100.0
```

### Ensure Processed Lead  — *hidden*

**Table:** Call Log · **Type:** add a new row to another table using values from this row · **Target:** Processed Leads

**Condition:**
```
COUNT(SELECT(Processed Leads[Lead ID],
  [Lead ID] = [_THISROW].[lead_id])) = 0
```

**Sets 60 columns.** 58 are copied one-for-one by dereferencing the ref column:

```
<Column Name> = [lead_id].[<Column Name>]
```

for each of: Lead ID, Phone Number, Country Code, Name, Email, Creation Date, Last Modified, Tags, User ID, Entry Channel, Channel Type, Status, Account Owner Name, Account Owner Email, Closure Due On, Deal Value, WhatsApp Opted In, Lead Source, Internal Lead Source, Campaign Name, Campaign ID, Course Name, Timeline, Time Slot, State, City, Call Disposition, Remarks, Company Name, Gender, Age in Years, Date of Birth, Date of Joining, Employee ID, Father's Name, Mother's Name, Qualification, Year of Passing, Years of Experience, Jobs, Work Specialisation, Current Salary, Expected Salary, Preferred City, Preferred State, Willing to Relocate, Current Pincode, Location, Rank, Source ID, Source URL, Marked as Spam, Stage Updated At, Stage UUID, Owner UUID, Lead Age (Days), Days to Closure, `_row_hash`.

Two are not copies:
```
Updated Status = [status_after_call]
Sync Error     = ""
```

> The target table must have `Require` **off** on every column this action does not set, or the add fails silently.

### Ask Status After Call  — *hidden*

**Table:** Call Log · **Type:** go to another view within this app

```
LINKTOROW([lead_id], "End Call Status Form")
```

### End Call  — *prominent*

**Table:** Call Log · **Type:** grouped

1. Set End Time
2. Ensure Processed Lead
3. Ask Status After Call

Order matters: the processed row must exist before the form that edits it is opened.

### Set Status After Call  — *hidden*

**Table:** Call Log · **Type:** set column values

```
status_after_call = LOOKUP([lead_id], "Processed Leads", "Lead ID", "Updated Status")
```

### Push Status To Call Log  — *hidden*

**Table:** Processed Leads · **Type:** execute an action on a set of rows

**Referenced table:** Call Log

**Referenced rows:**
```
SELECT(Call Log[call_log_id],
  AND([lead_id] = [_THISROW].[Lead ID],
      [call_end_time] = MAX(SELECT(Call Log[call_end_time],
                                   [lead_id] = [_THISROW].[Lead ID]))))
```

**Referenced action:** Set Status After Call

Targets only the most recent completed call. Historical rows keep the status that was true at the time.

### View Full Details  — *hidden*

**Table:** Leads Raw · **Type:** go to another view

```
LINKTOROW([Lead ID], "Lead Full Details")
```

### Open Filtered Leads  — *hidden*

**Table:** Statuses · **Type:** go to another view

```
LINKTOFILTEREDVIEW("My Leads", [Status] = [_THISROW].[status name])
```

---

## WMS pipeline

Same shape, with two differences worth noting.

### Start Call

```
lead_id            = [lead_id]
call_log_id        = UNIQUEID()
telecaller         = LOOKUP(USEREMAIL(), "Telecallers", "email", "wms_user_name")
call_start_time    = NOW()
status_before_call = [lead_status]
```

The telecaller is stored as a **name** resolved from email, not as the email. This is the root of the type-mismatch issues on this side; storing the email would have been better.

### Start Call & Dial

**Condition** — note the scope:
```
COUNT(SELECT(Call Log[call_log_id],
  AND(ISBLANK([call_end_time]),
      [telecaller] = LOOKUP(USEREMAIL(), "Telecallers", "email", "wms_user_name")))) = 0
```

Per *telecaller*, not per *lead*: one open call at a time across all leads. The Online version guards per lead. Both are defensible; per-lead is the one that survived.

### Sync Status From Latest Call

**Table:** Leads · **Type:** set column values

```
lead_status = LOOKUP(
  MAXROW("Call Log", "call_start_time", TEXT([lead_id]) = TEXT([_THISROW].[lead_id])),
  "Call Log", "call_log_id", "status_after_call")
```

`MAXROW` returns the key of the latest matching row; `LOOKUP` then reads a column from it. Note `TEXT()` on both sides — the keys differ in type across the two tables.

### Push Status to Lead

**Table:** Call Log · **Type:** execute an action on a set of rows

```
Referenced rows:   FILTER("Leads", TEXT([lead_id]) = TEXT([_THISROW].[lead_id]))
Referenced action: Sync Status From Latest Call
```

### Open Call Summary Detail

**Table:** Call Summary · **Type:** go to another view

```
IFS(
  [category_name] = "Status Changed",
    LINKTOFILTEREDVIEW("My Status Updates Today",
      AND(NOT(ISBLANK([status_after_call])),
          [status_after_call] <> [status_before_call])),
  [category_name] = "Status Not Changed",
    LINKTOFILTEREDVIEW("My Status Updates Today",
      OR(ISBLANK([status_after_call]),
         [status_after_call] = [status_before_call])))
```

### Send Daily Report

**Table:** Telecallers · **Type:** external — start an email

```
To:      INDEX(Managers[Manager Email], 1)
Subject: CONCATENATE("Call Report - ", USERNAME(), " - ",
                     TEXT(USERSETTINGS("report_start_date")), " to ",
                     TEXT(USERSETTINGS("report_end_date")))
```

Body computes total calls, total duration and placements within the date window.

> The recipient is hard-coded to the first row of the Managers table rather than resolved from the sender's own manager. A known defect.

---

## Meta pipeline (retired)

### Assign Lead

**Table:** Meta Leads Raw · **Type:** go to another view

```
LINKTOFORM("Leads_Form",
  "meta_lead_key", [meta_lead_key],
  "full_name",     [full_name],
  "phone_clean",   [meta_phone_clean],
  "campaign_name", [campaign_name],
  "campaign_id",   [campaign_id],
  "state",         [state],
  "city",          [city],
  "manager_name",  [meta_manager],
  "status",        "New Lead")
```

`LINKTOFORM` opens a prefilled form rather than writing directly — the manager confirms the assignment instead of it happening silently.

### Ensure Processed Lead Exists

Same pattern as the Online version, but resolves campaign and manager through a `LOOKUP` on the working table rather than a ref dereference, because the Meta working table is keyed on row number rather than on the lead id. Dereferencing is cleaner; key your tables properly and you can use it.
