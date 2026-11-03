# Views and slices

## The scoping pattern

Every "my …" view in the app is a **slice** — a filtered subset of a table — with the filter resolving against `USEREMAIL()`. There is no role column, no permissions table and no per-user configuration.

Two shapes cover everything:

**Direct — the table carries the user's email:**
```
[telecaller_email] = USEREMAIL()
```

**Indirect — the table carries a name, resolved through the roster:**
```
[wms_user_name] = LOOKUP(USEREMAIL(), "Telecallers", "Email", "wms_user_name")
```

**Manager scope — walk the roster graph:**
```
IN([wms_user_name],
   SELECT(Telecallers[wms_user_name],
     IN([Email],
        SELECT(Team Mapping[telecaller_email],
          [manager_id] = LOOKUP(USEREMAIL(), "Managers", "Manager Email", "Manager ID")))))
```

That last expression appears, with variations, in four places. It is the single most reused piece of logic in the system.

## Slices

15 slices. Exact filters in [`spec/expressions/slices.md`](../spec/expressions/slices.md).

| Slice | Table | Purpose |
|---|---|---|
| My Open Calls (Online) | Call Log | Calls started by me, not yet ended |
| My Leads | Leads Raw | Leads assigned to me |
| My Open Calls (WMS) | Call Log | As above, WMS side |
| My Assigned Leads | Leads | My WMS leads |
| My Today's Calls | Call Log | Today's activity, drives the summary tiles |
| My Report Calls | Call Log | My calls within the user-set date range |
| My Team | Team Mapping | My team, if I am a manager |
| My Team Calls | Call Log | Team activity within the date range |
| My Team Leads | Leads | Team's leads |
| My Team Quotas | Assignment Quotas | Quota bands I own |
| My Telecaller Profile | Telecallers | My own row |
| Meta Leads Clean | Meta Leads Raw | Test rows excluded |
| My Unassigned Meta Leads | Meta Leads Raw | Mine, clean, not yet assigned |
| My Meta Leads | Meta Leads Working | Assigned to me, unworked |
| My Meta Open Calls | Meta Call Log | Open Meta calls |

### Slice actions do not inherit

**This is the single most expensive gotcha in the system.** A slice does not inherit its parent table's custom actions. A view built on a slice shows no custom buttons at all unless each action is attached explicitly under the slice's own action list.

The symptom is silent: the view renders correctly, the data is right, and the button a user needs is simply absent. Every slice-backed view that needs a button needs that button added twice — once on the table, once on the slice.

## Views

Roughly 90 views, most of them the `_Detail` / `_Form` / `_Inline` set AppSheet generates per table.

### Navigation

AppSheet's bottom bar holds a maximum of **5** views. The app used 4:

| Position | View | Data | Type |
|---|---|---|---|
| first | WMS Lead Pipeline | Statuses | deck |
| next | WMS Open Calls | My Open Calls | deck |
| middle | Rank Pipeline | Ranks | deck |
| later | Online Open Calls | My Open Calls (Online) | deck |

Everything else sits at position `ref` — reachable only by navigating from another view, never from the bar.

### The pipeline decks

The two primary entry points are both decks over a *reference* table rather than the lead table, with a live count per row:

- **Status pipeline** — one row per status, count of leads in it, scoped to the user's own leads (or team, if a manager)
- **Rank pipeline** — one row per priority rank 0–5, count of leads at that rank

Tapping a row calls `LINKTOFILTEREDVIEW` to open the lead deck pre-filtered to that status or rank.

This is the design choice that makes the app usable at volume: a telecaller lands on *"Rank 1: 48 leads"*, not on an undifferentiated list of thousands.

```
Open Filtered Leads  =  LINKTOFILTEREDVIEW("My Leads",
                          [lead_status] = [_THISROW].[status name])
```

### The End Call Status form

A form view on the processed-lead table, at position `ref`, opened by `LINKTOROW` from the End Call action. It shows the fields a telecaller needs to fill or verify after a call — status, disposition, remarks, and the candidate profile fields — and hides every internal column: IDs, rank, account owner, tags, row hashes and all sync bookkeeping.

Which columns appear is governed by the `Show?` flag on the table, not by the form definition.

### Format rules

Two, both on the WMS side: a check icon on call log rows where the lead reached "Joined", and a star on leads in that status. The Online side has none — an inconsistency, not a decision.

### Reporting views

Manager-facing views on the WMS side only: Team Leaderboard, Team Report, Team Leads, My Report. All bounded by two user-settable dates (`report_start_date`, `report_end_date`) exposed through a Settings form.

The equivalent manager layer was never built for the Online pipeline.

## User settings

A single-row-per-user table exposing two fields: a report start date and a report end date. Referenced in slice filters and rollup formulas as `USERSETTINGS("report_start_date")`.

AppSheet warns that user-settings changes require an immediate sync. The warning is accurate and unavoidable — it is the cost of using settings as filter parameters.
