# Slice filters

Every slice filter, verbatim. All per-user scoping resolves against `USEREMAIL()`.

Remember: **slices do not inherit table actions.** Each slice that backs a view with buttons needs those actions attached under its own action list.

---

## Online pipeline

### My Open Calls
**Table:** Call Log
```
AND([telecaller_email] = USEREMAIL(),
    ISBLANK([call_end_time]))
```
**Columns:** all · **Slice action:** End Call · **Update mode:** updates + adds + deletes

The definition of "open" is a row with a start time and no end time. No status flag, nothing to get out of sync.

---

## WMS pipeline

### My Open Calls
**Table:** Call Log
```
AND(ISBLANK([call_end_time]),
    [telecaller] = LOOKUP(USEREMAIL(), "Telecallers", "email", "wms_user_name"))
```

Indirect resolution: this table stores a display name, so the email has to be translated through the roster first.

### My Assigned Leads
**Table:** Leads
```
[wms_user_name] = LOOKUP(USEREMAIL(), "Telecallers", "Email", "wms_user_name")
```

### My Today's Calls
**Table:** Call Log
```
AND([telecaller] = LOOKUP(USEREMAIL(), "Telecallers", "email", "wms_user_name"),
    [call_start_time] >= TODAY())
```

`>= TODAY()` compares a DateTime against a Date, which AppSheet coerces to midnight. Works, but `DATE([call_start_time]) = TODAY()` is clearer.

### My Report Calls
**Table:** Call Log
```
AND([telecaller] = LOOKUP(USEREMAIL(), "Telecallers", "email", "wms_user_name"),
    [call_start_time] >= USERSETTINGS("report_start_date"),
    [call_start_time] <= (USERSETTINGS("report_end_date") + 1))
```

The `+ 1` makes the end date inclusive. Without it, a DateTime at 14:30 on the end date falls outside a range bounded by that date at midnight — so the user's last day silently returns nothing.

### My Team
**Table:** Team Mapping
```
[manager_id] = LOOKUP(USEREMAIL(), "Managers", "Manager Email", "Manager ID")
```

Returns nothing for a telecaller, which is exactly the desired behaviour — the view is empty rather than forbidden.

### My Team Calls
**Table:** Call Log
```
AND(IN([telecaller],
        SELECT(Telecallers[wms_user_name],
          IN([Email],
             SELECT(Team Mapping[telecaller_email],
               [manager_id] = LOOKUP(USEREMAIL(), "Managers",
                                     "Manager Email", "Manager ID"))))),
    [call_date] >= USERSETTINGS("report_start_date"),
    [call_date] <= USERSETTINGS("report_end_date"))
```

The nested `SELECT`/`IN` walks Managers → Team Mapping → Telecallers. This is the most reused expression in the system; it appears in four places with minor variations.

Note it filters on `call_date` (a date virtual column) rather than `call_start_time`, so no `+ 1` is needed here.

### My Team Leads
**Table:** Leads
```
IN([wms_user_name],
   SELECT(Telecallers[wms_user_name],
     IN([Email],
        SELECT(Team Mapping[telecaller_email],
          [manager_id] = LOOKUP(USEREMAIL(), "Managers",
                                "Manager Email", "Manager ID")))))
```

### My Team Quotas
**Table:** Assignment Quotas
```
[manager_name] = LOOKUP(USEREMAIL(), "Managers", "Manager Email", "Manager Name")
```
**Slice action:** Auto assign

Resolves on manager *name*, not ID, because the quota table stores the name. Inconsistent with the slices above; storing the ID everywhere would be better.

### My Telecaller Profile
**Table:** Telecallers
```
[email] = USEREMAIL()
```

A single-row slice, used as the anchor for the daily-report email action.

---

## Meta pipeline (retired)

### Meta Leads Clean
**Table:** Meta Leads Raw
```
NOT([is_test_row])
```

### My Unassigned Meta Leads
**Table:** Meta Leads Raw
```
AND(NOT([is_test_row]),
    NOT([already_assigned]),
    [meta_manager] = LOOKUP(USEREMAIL(), "Managers", "Manager Email", "Manager Name"))
```

Three conditions composed from virtual columns, which keeps the filter readable. Inlining the test-row heuristics here would have made it unmaintainable.

### My Meta Leads
**Table:** Meta Leads Working
```
AND([assigned_telecaller] = USEREMAIL(),
    ISBLANK([Status]))
```

`assigned_telecaller` is itself computed — see [`virtual-columns.md`](virtual-columns.md). The assignment is not stored anywhere; it is derived on read, so changing a quota band reassigns leads immediately.

### My Meta Open Calls
**Table:** Meta Call Log
```
AND([telecaller_email] = USEREMAIL(),
    ISBLANK([call_end_time]))
```

---

## Patterns

**Direct** — the table stores the email:
```
[telecaller_email] = USEREMAIL()
```

**Indirect** — the table stores a name:
```
[name_column] = LOOKUP(USEREMAIL(), "Telecallers", "Email", "wms_user_name")
```

**Manager scope** — walk the roster graph:
```
IN([name_column],
   SELECT(Telecallers[wms_user_name],
     IN([Email],
        SELECT(Team Mapping[telecaller_email],
          [manager_id] = LOOKUP(USEREMAIL(), "Managers",
                                "Manager Email", "Manager ID")))))
```

**Open record** — a start with no end:
```
ISBLANK([call_end_time])
```

Prefer the direct form. The indirect form exists only because an early table stored a display name instead of an email, and every lookup since has had to pay for it.
