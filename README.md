# AppSheet Lead Management CRM

A production lead-management CRM for a staffing and recruitment company's telecalling teams, built end to end on **Google AppSheet + Google Sheets** by a single analyst, with no engineering team and no budget line.

This repository is the **specification** of that system — every table, column type, expression, action, slice and security rule — written down so the app can be audited, handed over, or rebuilt from scratch by someone who has never seen it.

> **No real data here.** All spreadsheet IDs, app IDs, email addresses, names, report URLs and record counts have been replaced with placeholders. The logic is real; the identifiers are not. See [Anonymisation](#anonymisation).

---

## The problem

Telecallers were working leads out of shared spreadsheets. That meant:

- no record of who called whom, or when, or for how long
- lead status updated by whoever opened the sheet last, with no audit trail
- managers with no way to see team activity without asking for it
- two separate lead sources (an offline recruitment system and paid social ads) handled with completely different manual processes

The requirement was a CRM that a telecaller could run from a phone, that logged every call automatically, and that cost nothing beyond the Google Workspace licences the company already had.

## The approach

AppSheet sits directly on top of Google Sheets. No servers, no deployment pipeline, no database migration. The Sheets stay the system of record — which matters, because the upstream lead feeds and the downstream reporting both already lived in Sheets.

The whole thing is declarative: tables, column types, expressions, actions, slices. The "code" in this repo is AppSheet expression language.

## Architecture

Three lead pipelines were built over time. Two are retained; one was deliberately retired.

```
                    ┌─────────────────────────────────────────┐
                    │     Google Sheet workbook (one file)    │
                    │           — system of record —          │
                    └─────────────────────────────────────────┘
                       │              │                │
         ┌─────────────┘              │                └──────────────┐
         ▼                            ▼                               ▼
   ┌───────────┐              ┌──────────────┐               ┌──────────────┐
   │   WMS     │              │    Online    │               │     Meta     │
   │ (offline) │              │  (WhatsApp   │               │  (paid ads)  │
   │   LOCKED  │              │   CRM feed)  │               │   RETIRED    │
   └───────────┘              │     LIVE     │               └──────────────┘
                              └──────────────┘
```

| Pipeline | Source | Status | Why |
|---|---|---|---|
| **WMS / Offline** | Internal recruitment system export | Locked — working, frozen | First build. Full manager reporting layer. No further changes. |
| **Online** | WhatsApp business CRM, synced into the Sheet | **Live** | Current production pipeline. ~60-field lead records. |
| **Meta / Ads** | Meta Lead Ads | Retired, hidden | Built, worked, then abandoned when ad spend moved. Kept documented — the campaign→manager routing logic is reusable. |

Full detail: [`docs/01-architecture.md`](docs/01-architecture.md)

## The call flow

The core loop. Everything else is scaffolding around this:

```
  Telecaller opens a lead
        │
        ▼
  [Start Call & Dial]  ──► writes a Call Log row (start time, status before call)
        │                  ──► opens the phone dialler
        │
        │   guard: a second Start Call on the same lead is blocked
        │          while the first call is still open
        ▼
  ... call happens ...
        │
        ▼
  [End Call]  ──► stamps end time, computes duration
              ──► creates the Processed Lead row if it doesn't exist
                  (copies 58 fields from the raw lead, once only)
              ──► opens the End Call Status form
        │
        ▼
  Telecaller picks status / disposition / remarks and saves
        │
        ▼
  Status is pushed back onto the latest Call Log row
```

Two buttons for the telecaller. Seven hidden actions doing the work behind them.

Full detail: [`docs/03-call-flow.md`](docs/03-call-flow.md) · exact expressions in [`spec/expressions/actions.md`](spec/expressions/actions.md)

## Design decisions worth calling out

**Raw and processed leads are separate tables.** The raw table is a mirror of the upstream feed — written by the sync, never by a human. Telecallers write to a processed table that is created on first contact. The sync can overwrite the raw table at any time without destroying a telecaller's work.

**Per-user scoping is done with slices, not permissions.** Every "my leads / my open calls" view is a slice filtered on `USEREMAIL()`. Manager scope resolves through a Managers → Team Mapping → Telecallers chain, also by email. One expression pattern, reused everywhere.

**Role detection has no role column.** A user is a manager if their email resolves in the Managers table, and a telecaller otherwise:

```
ISBLANK(LOOKUP(USEREMAIL(), "Managers", "Manager Email", "Manager ID"))
```

**Row-level security is enforced at the table, not the view.** The raw leads table carries a security filter `[Account Owner Email] = USEREMAIL()`, so a telecaller cannot reach another telecaller's leads even by URL.

**Duplicate work is prevented with count guards**, not with flags that can drift:

```
COUNT(SELECT(Call Log[call_log_id],
  AND(ISBLANK([call_end_time]), [lead_id] = [_THISROW].[lead_id]))) = 0
```

Full detail: [`docs/05-security-roles.md`](docs/05-security-roles.md)

## What went wrong

Documented honestly in [`docs/lessons-learned.md`](docs/lessons-learned.md). The short version:

- **Slice views do not inherit table actions.** Buttons silently vanish. Add them under the slice's own action list.
- **Changing a column's type must be done on every table that references it.** Miss one and the ref breaks at runtime, not at save time.
- **A required column on the target table silently breaks an automated row-copy action.** Keep `Require` off on anything the automation writes.
- **Never rename a column that an expression references** — change the display name instead. AppSheet does not refactor for you.
- **Hiding a tab in Google Sheets does not hide it from AppSheet.**
- **The free-plan user cap is per app, counted by monthly active users** — and it counts anyone who opens the app, not just the people on the user list. This is what eventually forced the pipeline split documented in [`docs/06-rebuild.md`](docs/06-rebuild.md).

## Repository layout

```
.
├── docs/
│   ├── 01-architecture.md     Pipelines, data flow, table inventory
│   ├── 02-data-model.md       Tables, columns, types, keys, refs
│   ├── 03-call-flow.md        The call loop, step by step
│   ├── 04-views-slices.md     Navigation, slices, per-user scoping
│   ├── 05-security-roles.md   Sign-in, security filters, role detection
│   ├── 06-rebuild.md          Splitting the live pipeline into its own app
│   └── lessons-learned.md     Mistakes, costs, rules derived from them
└── spec/
    ├── enums.md               Status vocabulary, dispositions, ranks
    ├── tables/                One file per retained table
    └── expressions/
        ├── actions.md         Exact action definitions
        ├── slices.md          Exact slice filter expressions
        └── virtual-columns.md Computed column formulas
```

## Rebuilding from this spec

1. Create the Google Sheet tabs listed in [`docs/02-data-model.md`](docs/02-data-model.md).
2. Create an AppSheet app from that Sheet.
3. Set column types and flags per [`spec/tables/`](spec/tables/) — types first, everything else depends on them.
4. Create the reference tables and their refs before any expression that uses them.
5. Add actions in dependency order: setters, then the row-copy, then the grouped actions.
6. Add slices, then attach slice actions explicitly.
7. Apply the security filter and user list.
8. Run the test checklist at the end of [`docs/06-rebuild.md`](docs/06-rebuild.md).

## Anonymisation

Replaced throughout:

| Real value | Placeholder |
|---|---|
| Google Sheet file IDs | `<SHEET_ID>` |
| AppSheet app IDs | `<APP_ID>` |
| Company name | "the company" / generic |
| Colleague and manager names | "the sync owner", "Manager A" |
| Email addresses | `telecaller@example.com`, `manager@example.com` |
| BI report URLs | removed entirely |
| Lead volumes, user counts | removed or generalised |

Table and column names are retained — they are generic CRM vocabulary (`lead_id`, `call_duration_min`, `status_after_call`) and carry no business data.

## Licence

MIT — see [LICENSE](LICENSE). Applies to the documentation and expressions in this repository.
