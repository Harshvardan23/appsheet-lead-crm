# Reference tables

Five lookup tables. All read-only in the app, all keyed and labelled on a single value column.

---

## Ranks

Priority buckets 0–5. The most important reference table, because it backs the app's primary navigation view.

| Column | Type | Flags |
|---|---|---|
| rank_value | Number | **K**, *hidden* |
| Rank Label | Text | **L**, *noEdit*, virtual |
| Lead Count | Number | *noEdit*, virtual |
| Related Leads Raw | List | *noEdit*, virtual |
| Related Processed Leads | List | *noEdit*, virtual |

```
Rank Label = CONCATENATE("Rank ", [rank_value])

Lead Count = COUNT(SELECT(Leads Raw[Lead ID],
  OR([Rank] = [_THISROW].[rank_value],
     AND(ISBLANK([Rank]), [_THISROW].[rank_value] = 0))))

Related Leads Raw = SELECT(Leads Raw[Lead ID],
  OR([Rank] = [_THISROW].[rank_value],
     AND(ISBLANK([Rank]), [_THISROW].[rank_value] = 0)))

Related Processed Leads = REF_ROWS("Processed Leads", "Rank")
```

The blank-equals-zero clause appears twice and must match in both. A lead with no rank is unreachable without it — not hidden, not zero, simply absent from every rank bucket.

---

## Call Dispositions

| Column | Type | Flags |
|---|---|---|
| disposition_value | Text | **K**, **L**, REQ |
| Related Processed Leads | List | *noEdit*, `f=REF_ROWS("Processed Leads", "Call Disposition")` |

16 values. Distinct from lead status: disposition records what happened on *this call*, status records where the lead now stands. One lead can sit at "Follow up" across several calls with a different disposition each time.

---

## Qualifications

| Column | Type | Flags |
|---|---|---|
| qualification_value | Text | **K**, **L**, REQ |
| Related Processed Leads | List | *noEdit*, `f=REF_ROWS("Processed Leads", "Qualification")` |

A table rather than an Enum so a manager can extend it in the app without an editor change.

---

## India States

| Column | Type | Flags |
|---|---|---|
| state_name | Text | **K**, **L**, REQ |
| Related Processed Leads | List | *noEdit*, `f=REF_ROWS("Processed Leads", "State")` |
| Related by Preferred State | List | *noEdit*, `f=REF_ROWS("Processed Leads", "Preferred State")` |

28 states + 8 union territories. Referenced twice from the lead record — current and preferred — so relocation willingness is a structured comparison rather than free text.

---

## India Cities

| Column | Type | Flags |
|---|---|---|
| `_RowNumber` | Number | **K** — *see below* |
| City_name | Text | **L** |
| Related Processed Leads | List | *noEdit*, `f=REF_ROWS("Processed Leads", "City")` — **stale** |
| Related by Preferred City | List | *noEdit*, `f=REF_ROWS("Processed Leads", "Preferred City")` |

> **Two defects, both instructive.**
>
> **Keyed on row number.** A row-number key is positional: insert or delete a row in the Sheet and every key below it shifts, silently repointing every ref that targeted them. Key a reference table on its value column.
>
> **A stale `REF_ROWS`.** The first virtual column targets `Processed Leads.City`, which was converted from `Ref` to `Text`. `REF_ROWS` binds by column name and does not validate the target's type — so it saves cleanly, raises no error, and returns nothing. This is the canonical example of AppSheet failing silently.

---

## Statuses (legacy)

| Column | Type | Flags |
|---|---|---|
| status_name | Text | **K**, **L**, REQ |
| Lead Count | Number | *noEdit*, `f=COUNT(SELECT(Leads Raw[Lead ID], [Status] = [_THISROW].[status name]))` |

13 pipeline stages. Backed the original status-based navigation deck, superseded by the rank pipeline. Retained but off the navigation bar.

---

## Build rule

Create every reference table and set its key and label **before** converting any column to a `Ref` pointing at it. AppSheet will not offer a table as a ref target until it has a key, and a ref created against a table whose key later changes does not follow.
