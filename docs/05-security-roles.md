# Security and roles

## Authentication

| Setting | Value |
|---|---|
| Require user sign-in | ON |
| Authentication provider | Google |
| Allow all signed-in users | **OFF** |
| Access model | Explicit user list |

"Allow all signed-in users" being off is the important one. With it on, anyone with a Google account and the app link can open the app. Off, access is an explicit allowlist maintained in the app's user management.

This also has a billing consequence — see [06-rebuild.md](06-rebuild.md).

## Row-level security

AppSheet applies **security filters** at the table level, before any view or slice is evaluated. A row excluded by a security filter is not downloaded to the device at all.

The live lead table carries:

```
[Account Owner Email] = USEREMAIL()
```

A telecaller's device holds only the leads assigned to them upstream. Another telecaller's leads are not hidden — they are absent. They cannot be reached by deep link, by search, or by inspecting local storage.

Every other table is set to "access as app creator" with no filter. Those tables are either reference data (states, cities, dispositions) or are scoped at the view layer by slices.

### Security filter vs slice filter

| | Security filter | Slice filter |
|---|---|---|
| Evaluated | Server-side, before sync | Client-side, after sync |
| Excluded rows | Never leave the server | Present on the device |
| Use for | Confidentiality | Convenience and navigation |

Slices are a usability feature, not a security boundary. Anything that genuinely must not reach a user needs a security filter.

## Role detection

There is no role column. A user is a manager if and only if their email resolves in the Managers table:

```
ISBLANK(LOOKUP(USEREMAIL(), "Managers", "Manager Email", "Manager ID"))
  → TRUE   = telecaller
  → FALSE  = manager
```

Used inline wherever behaviour differs by role. The clearest example is the pipeline count, which returns the user's own leads for a telecaller and the whole team's for a manager:

```
IF(ISBLANK(LOOKUP(USEREMAIL(), "Managers", "Manager Email", "Manager ID")),

   COUNT(SELECT(Leads[lead_id],
     AND([lead_status] = [_THISROW].[status name],
         [wms_user_name] = ANY(SELECT(Telecallers[wms_user_name],
                                      [email] = USEREMAIL()))))),

   COUNT(SELECT(Leads[lead_id],
     AND([lead_status] = [_THISROW].[status name],
         IN([wms_user_name],
            SELECT(Telecallers[wms_user_name],
              IN([Email],
                 SELECT(Team Mapping[telecaller_email],
                   [manager_id] = LOOKUP(USEREMAIL(), "Managers",
                                         "Manager Email", "Manager ID")))))))))
```

One deck view serves both roles. No duplicate views, no conditional navigation.

### Why no role column

A role column has to be maintained, and goes stale when someone is promoted or leaves. Deriving the role from membership in the Managers table means there is exactly one place to update — the same place you would update anyway to add someone to the reporting chain.

The cost is that the role check is an expression, repeated inline, and a change to the roster shape means editing it in several places.

## The role graph

```
          USEREMAIL()
               │
               ├──► Managers        (Manager Email → Manager ID)
               │         │
               │         └──► Team Mapping   (manager_id → telecaller_email)
               │                   │
               └──► Telecallers ◄──┘          (Email → wms_user_name)
                         │
                         └──► Leads / Call Log  (by wms_user_name)
```

Everything resolves by email. Nothing resolves by display name except the final hop, and that hop is the source of the one type-mismatch bug documented in [02-data-model.md](02-data-model.md).

## Other security settings

| Setting | State | Note |
|---|---|---|
| Encrypt device data | OFF | Default |
| Secure PDF access | OFF | Default |
| Secure image access | OFF | Default |
| Require image and file URL signing | ON | Default |

Default posture, not a reviewed decision. For an app holding candidate PII — names, phone numbers, salary expectations — device encryption is worth turning on, and is noted as an open item.

## Open issues

- **Device encryption is off.** The app holds personally identifying candidate data on telecaller phones.
- **No security filter on the processed-lead table.** Only the raw table is filtered. A telecaller can in principle reach processed rows for leads they do not own.
- **No audit of who changed a lead's status**, beyond the call log. A status edited directly on the processed lead, outside the call flow, leaves no trace.
- **The reporting email action hard-codes its recipient** to the first row of the Managers table (`INDEX(Managers[Manager Email], 1)`) rather than resolving the sender's own manager.
