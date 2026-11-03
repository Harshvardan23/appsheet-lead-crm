# The call flow

The core loop of the system. A telecaller sees two buttons; seven actions run behind them.

## Sequence

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. Telecaller opens a lead from the Rank Pipeline                    │
└──────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. [Start Call & Dial]              ← PROMINENT, on Leads Raw        │
│                                                                      │
│    a. Start Call                                                     │
│       GUARD: no open call already exists for this lead               │
│       adds a Call Log row:                                           │
│         call_log_id       ← UNIQUEID()                               │
│         lead_id           ← [Lead ID]                                │
│         full_name         ← [Name]                                   │
│         phone_number      ← [Phone Number]                           │
│         telecaller_email  ← USEREMAIL()                              │
│         call_start_time   ← NOW()                                    │
│         status_before_call← [Status]                                 │
│                                                                      │
│    b. Call Phone (system action, on [Phone Number])                  │
│       hands off to the device dialler                                │
└──────────────────────────────────────────────────────────────────────┘
                                 │
                        ... the call happens ...
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. [End Call]                       ← PROMINENT, on Call Log         │
│    reached from the "My Open Calls" view                             │
│                                                                      │
│    a. Set End Time                                                   │
│         call_end_time     ← NOW()                                    │
│         call_duration_min ← ROUND((TOTALSECONDS(NOW() -              │
│                               [call_start_time]) / 60.0) * 100)/100.0│
│                                                                      │
│    b. Ensure Processed Lead                                          │
│         GUARD: no processed row exists for this lead                 │
│         copies 58 fields from Leads Raw via the ref column           │
│         + Updated Status ← [status_after_call]                       │
│         + Sync Error     ← ""                                        │
│                                                                      │
│    c. Ask Status After Call                                          │
│         LINKTOROW([lead_id], "End Call Status Form")                 │
└──────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 4. Telecaller fills the End Call Status form on the processed lead   │
│    Status After Call · Call Disposition · Remarks · profile fields   │
└──────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 5. Push Status To Call Log         ← on Processed Leads              │
│    targets ONLY the most recent call log row for this lead:          │
│      SELECT(Call Log[call_log_id],                                   │
│        AND([lead_id] = [_THISROW].[Lead ID],                         │
│            [call_end_time] = MAX(SELECT(Call Log[call_end_time],     │
│              [lead_id] = [_THISROW].[Lead ID]))))                    │
│    runs Set Status After Call on it:                                 │
│      status_after_call ← LOOKUP([lead_id], "Processed Leads",        │
│                                 "Lead ID", "Updated Status")         │
└──────────────────────────────────────────────────────────────────────┘
```

Exact definitions: [`spec/expressions/actions.md`](../spec/expressions/actions.md)

## Why it is built this way

### Two prominent buttons, seven hidden actions

Telecallers are working at volume on a phone. Every visible button is a chance to tap the wrong one. The setters, the row-copy and the form navigation are all set to **Hide** — they only ever run as members of a grouped action. The telecaller sees **Start Call & Dial** and **End Call** and nothing else.

### The open-call guard

```
COUNT(SELECT(Call Log[call_log_id],
  AND(ISBLANK([call_end_time]),
      [lead_id] = [_THISROW].[Lead ID]))) = 0
```

An unfinished call is defined as a row with a start time and no end time. The guard makes Start Call unavailable while one exists. Without it, a telecaller who taps twice — or whose connection stutters — produces two open rows, and duration data becomes meaningless.

The WMS variant of this guard is scoped per *telecaller* rather than per *lead*: a caller may only have one call open at a time across all leads. Both are defensible; the per-lead version is the one that survived.

### Duration computed, never entered

`TOTALSECONDS(NOW() - [call_start_time])` divided by 60, rounded to two decimals. Nobody types a duration, so nobody rounds a four-minute call up to ten.

### The row-copy, and why it is 58 fields

The processed lead has to be a complete, standalone record — reporting reads it directly, and the raw table may be rewritten by the sync at any time. So the copy is exhaustive rather than selective. Each field is set as `[lead_id].[<same column name>]`, dereferencing through the ref column.

This is the single most tedious part of the build and the main argument for copying an existing app rather than rebuilding it by hand.

The guard means it runs exactly once per lead. On the second and subsequent calls, the processed row already exists and the action is a no-op.

### Status flows both ways

The telecaller sets status on the **processed lead** (that is where the form is). The **call log** needs the same value, so the outcome of each individual call is recorded against that call rather than only against the lead.

`Push Status To Call Log` resolves the latest call row by `MAX(call_end_time)` and copies the value across. Historical call rows are never touched.

## Failure modes

| Symptom | Cause |
|---|---|
| Start Call button missing | Viewing through a slice; slice actions must be attached explicitly |
| Dialler does not open | Phone column type reverted to `Text` |
| Duplicate processed rows | Count guard missing or referencing the wrong key column |
| Row-copy silently does nothing | A `Require` column on the target table is unset |
| status_after_call stays blank | Push Status ran before the form was saved, or the Enum column was renamed |
