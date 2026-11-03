# Table specifications

One file per retained table of the live pipeline — the column list with the exact AppSheet flags needed to rebuild it.

| File | Table | Role |
|---|---|---|
| [leads-raw.md](leads-raw.md) | Leads Raw | Upstream mirror, read-only to humans, security-filtered |
| [processed-leads.md](processed-leads.md) | Processed Leads | Telecaller-owned outcome record |
| [call-log.md](call-log.md) | Call Log | One row per call attempt |
| [reference-tables.md](reference-tables.md) | Ranks, Dispositions, Qualifications, States, Cities | Lookups |

Narrative context for all of these is in [`docs/02-data-model.md`](../../docs/02-data-model.md).

## Notation

| Mark | Meaning |
|---|---|
| **K** | Key column |
| **L** | Label column |
| **REQ** | Require? ON |
| *hidden* | Show? OFF |
| *noEdit* | Editable? OFF |
| `i=` | Initial value |
| `f=` | App formula or virtual column formula |

## Build order

Types first, then refs, then formulas. A ref cannot be created before its target table exists, and a formula referencing a column of the wrong type saves without error and fails at runtime.

1. Create all tables from their Sheet tabs
2. Set keys and labels
3. Set base types on every column
4. Create the reference tables' key/label columns
5. Convert ref columns to `Ref` and point them at their targets
6. Add virtual columns and formulas
7. Set Show / Editable / Require flags
8. Apply security filters
