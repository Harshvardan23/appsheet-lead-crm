# Virtual columns

Computed columns, evaluated on read. They cost sync time on large tables, so each one here earns its place.

---

## Online pipeline

### Ranks — Lead Count
```
COUNT(SELECT(Leads Raw[Lead ID],
  OR([Rank] = [_THISROW].[rank_value],
     AND(ISBLANK([Rank]), [_THISROW].[rank_value] = 0))))
```

The blank-rank clause is essential. Without it, unranked leads are counted nowhere and reachable from nowhere.

### Ranks — Rank Label
```
CONCATENATE("Rank ", [rank_value])
```

Generated rather than stored, so a new rank value needs no label maintenance.

### Ranks — Related Leads
```
SELECT(Leads Raw[Lead ID],
  OR([Rank] = [_THISROW].[rank_value],
     AND(ISBLANK([Rank]), [_THISROW].[rank_value] = 0)))
```

Same predicate as the count. A hand-written `SELECT` rather than `REF_ROWS` precisely because `REF_ROWS` cannot express the blank-equals-zero rule.

### Statuses — Lead Count
```
COUNT(SELECT(Leads Raw[Lead ID], [Status] = [_THISROW].[status name]))
```

### Processed Leads — Status Before Call
```
[Status]
```

Captures the inbound status as a separate read-only field, so the form can show "was → now" without the user having to remember.

### Reference tables — related rows
```
REF_ROWS("Processed Leads", "<ref column name>")
```

On States, Cities, Dispositions and Qualifications. Note that `REF_ROWS` binds to the column by name and silently returns nothing if that column's type stops being a `Ref` — which has happened here, on the city reference.

---

## WMS pipeline

### Call Log — Status_changed
```
AND(NOT(ISBLANK([status_after_call])),
    [status_after_call] <> [status_before_call])
```

Drives the day's summary split: calls that moved a lead versus calls that did not.

### Call Log — call_date
```
DATE([call_start_time])
```

A date-only column makes range filters correct without the inclusive-end-date `+ 1` workaround.

### Call Log — lead_phone
```
LOOKUP([lead_id], "Leads", "lead_id", "mobile_number")
```

> **Broken.** Returns blank. `lead_id` is `Number` on Leads and `Text` on Call Log and the lookup does not coerce. Fix: `TEXT()` both sides. Left as-is because this side is frozen.

### Leads — _ComputedName
```
CONCATENATE([first_name], " ", [last_name])
```

The table's label column. Keeping the parts separate in storage and composing for display is the right way round.

### Team Mapping — rollups
```
telecaller_name = LOOKUP([telecaller_email], "Telecallers", "Email", "wms_user_name")

total_calls     = COUNT(SELECT(Call Log[call_log_id],
                    AND([telecaller] = [telecaller_name],
                        [call_date] >= USERSETTINGS("report_start_date"),
                        [call_date] <= USERSETTINGS("report_end_date"))))

total_duration  = SUM(SELECT(Call Log[call_duration_min],
                    AND([telecaller] = [telecaller_name],
                        [call_date] >= USERSETTINGS("report_start_date"),
                        [call_date] <= USERSETTINGS("report_end_date"))))

joined_count    = COUNT(SELECT(Call Log[call_log_id],
                    AND([telecaller] = [telecaller_name],
                        [call_date] >= USERSETTINGS("report_start_date"),
                        [call_date] <= USERSETTINGS("report_end_date"),
                        [status_after_call] = "Joined")))
```

A team leaderboard with no aggregation job, no scheduled rollup and no reporting table. Three virtual columns over the call log, bounded by two user-settable dates. The manager changes the date range and the whole leaderboard recomputes.

Each is a full scan of the call log, so this does not scale indefinitely — at some volume these need to become a materialised summary. That threshold had not been reached.

### Call Summary — Call Count
```
COUNT(SELECT(My Todays Calls[call_log_id],
  IF([category_name] = "Status Changed",
     [status_changed] = TRUE,
     [status_changed] = FALSE)))
```

A `SELECT` over a **slice** rather than a table, so the per-user scoping is inherited rather than repeated.

---

## Meta pipeline (retired)

The most interesting formulas in the system. Round-robin assignment with no automation.

### Leads Raw — meta_lead_key
```
CONCATENATE([id], "|", [phone_number], "|", [created_time])
```

Composite key. The platform's own id proved insufficient on its own for deduplication.

### Leads Raw — phone_clean
```
RIGHT(SUBSTITUTE(SUBSTITUTE([phone_number], "p:", ""), "+", ""), 10)
```

Strips the platform's `p:` prefix and the `+`, keeps the last 10 digits. Normalises every inbound format to a diallable Indian number.

### Leads Raw — is_test_row
```
OR(CONTAINS(UPPER([campaign_name]), "TEST"),
   CONTAINS(UPPER([campaign_name]), "SAMPLE"),
   CONTAINS(UPPER([form_name]),     "TEST"),
   CONTAINS(UPPER([form_name]),     "SAMPLE"),
   CONTAINS(UPPER([full_name]),     "TEST"))
```

A heuristic, not a guarantee — but ad platforms generate test submissions constantly, and a telecaller phoning one wastes real time. Imperfect filtering beats none.

### Leads Raw — already_assigned
```
IN([meta_lead_key], Leads[meta_lead_key])
```

### Leads Working — meta_manager
```
LOOKUP([campaign_name], "Campaign Manager Mapping", "campaign_name", "manager_name")
```

The whole ad-routing configuration is one two-column table, editable in the app. Launch a campaign, add a row, leads route. No editor change, no redeploy.

### Leads Working — lead_rank
```
COUNT(SELECT(Leads Working[id],
  AND([meta_manager] = [_THISROW].[meta_manager],
      ISBLANK([Status]),
      [created_time] >= [_THISROW].[created_time])))
```

A lead's position in its manager's queue of unworked leads, oldest first. Counts how many unworked leads of the same manager are at least as recent as this one.

Note `ISBLANK([Status])` — worked leads drop out of the queue, so the ranks of everything behind them shift up automatically.

### Leads Working — assigned_telecaller
```
ANY(SELECT(Assignment Quotas[telecaller_email],
  AND([manager_name] = [_THISROW].[meta_manager],
      [range_start]  <= [_THISROW].[lead_rank],
      [range_end]    >= [_THISROW].[lead_rank])))
```

Finds the quota band containing this lead's queue position and returns the telecaller who owns it.

**Why this is the nicest thing in the system.** Assignment is never written anywhere. It is computed on read, so:

- changing a quota band reassigns leads immediately, with no reassignment job
- a telecaller going on leave is handled by editing two numbers
- there is no assignment state to drift, repair or reconcile
- nothing has to run on a schedule, and nothing can fail overnight

The trade-off is that assignment history is not retained — you cannot ask who a lead *was* assigned to last week. For this workflow that was acceptable. For one with commission attached, it would not be.
