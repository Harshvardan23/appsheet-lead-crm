# Architecture

## Platform

| Layer | Technology |
|---|---|
| Data store | Google Sheets (one workbook, one tab per table) |
| Application | Google AppSheet |
| Authentication | Google sign-in, explicit user list |
| Client | AppSheet mobile app (Android/iOS) and browser |
| Reporting | External BI tool reading the same Sheet |

Every table in the app uses the same datasource: one Google Sheet workbook, with the table qualifier being the tab name. There is no separate database, no API layer and no build step.

## Why this stack

The constraint was zero incremental cost and zero engineering support. Google Workspace was already licensed. The upstream lead feeds already landed in Sheets, and the downstream BI dashboards already read from Sheets. Putting the CRM on the same substrate meant no integration work at either end — the app writes to the same cells the dashboards already read.

The trade-off, which showed up later, is AppSheet's per-app pricing by monthly active user. See [06-rebuild.md](06-rebuild.md).

## The three pipelines

The app accumulated three lead pipelines over its life. They share the workbook and the telecaller roster but have separate tables, actions and views.

### WMS / Offline — locked

The original build. Leads come from the company's internal recruitment system as a flat export.

```
Internal recruitment export
        │
        ▼
   Leads (37 columns)
        │
        ├──► Call Log ──► Call Summary (today's calls, split by status changed / not changed)
        │
        └──► Statuses (pipeline counts, scoped by role)

   Supporting: Managers, Telecallers, Team Mapping, Assignment Quotas
```

This side has the most complete management layer: a team leaderboard, per-telecaller reports bounded by user-settable date ranges, and a daily report email action. It works and is frozen. No changes are made to it.

### Online — live

Current production. Leads originate in a third-party WhatsApp business CRM, owned by a colleague, and are synced into the workbook.

```
Upstream WhatsApp CRM  ──(sync)──►  Leads Raw  (mirror; never written by a human)
                                        │
                                        │  first contact creates the row
                                        ▼
                                  Processed Leads  (telecaller writes outcomes here)
                                        │
                                        ▼
                                     Call Log  (one row per call attempt)

   Lookups: Ranks, Call Dispositions, Qualifications, States, Cities
```

The raw/processed split is the key decision. The raw table is a mirror of upstream — the sync may rewrite it at any time. Telecaller work lives in a separate table keyed on the same lead ID, created on first contact by an automated row-copy that duplicates 58 fields. Nothing a telecaller types can be clobbered by a sync.

Leads are prioritised by a numeric **Rank** (0–5) assigned upstream. The main navigation view is a deck of ranks with live lead counts, so a telecaller works top-rank-first rather than top-of-list-first.

### Meta / Paid ads — retired

Built for Meta Lead Ads, used, then retired when ad spend moved elsewhere. Kept in the repo because the routing logic is the most interesting part of the whole system and is reusable.

```
Meta Lead Ads webhook ──► Leads Raw
                              │
                              │  test rows filtered out by name heuristics
                              │  campaign name → manager lookup
                              ▼
                        Leads Working
                              │
                              │  lead_rank = position in that manager's unworked queue
                              │  assigned_telecaller = quota band containing that rank
                              ▼
                      Leads / Processed Leads ──► Call Log
```

Round-robin assignment with no automation and no server: a virtual column counts a lead's position in its manager's unworked queue, and a second virtual column looks up which telecaller owns that position range in an Assignment Quotas table. Change the quota rows and the distribution changes immediately, with no reassignment job to run.

See [`spec/expressions/virtual-columns.md`](../spec/expressions/virtual-columns.md) for the exact formulas.

## Table inventory

25 tables at the point this specification was taken.

| Group | Tables |
|---|---|
| Online (live) | Leads Raw, Processed Leads, Call Log, Ranks, Call Dispositions, Qualifications, Statuses |
| WMS (locked) | Leads, Call Log, Call Summary, Statuses, Team Mapping, Managers, Telecallers, Assignment Quotas |
| Meta (retired) | Leads Raw, Leads Working, Leads, Processed Leads, Call Log, Statuses, Campaign Manager Mapping |
| Shared lookups | India States, India Cities |
| Other | README (in-app help text) |

Largest tables by column count: Online Processed Leads (67), Online Leads Raw (61), WMS Leads (37), Meta Leads Working (31).

## Data flow guarantees

**The Sheet is the system of record.** The app is a client. Any process that can write to the Sheet — a sync script, a formula, a human with edit access — is part of the system and has to be reasoned about.

**Raw tables are read-only to humans.** Enforced by column-level `Editable_If = FALSE` rather than the base editable flag, so automation can still write them.

**Processed rows are created exactly once.** The row-copy action carries a count guard, so a second call on the same lead does not create a second processed row.

**Call logs are append-only in practice.** Nothing updates a historical call row except the status push, which only ever touches the most recent one.
