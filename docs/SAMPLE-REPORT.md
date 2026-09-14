# Sample output

What the agent hands back at the end of a run. This example is illustrative — a fictional ticket
against a fictional app — but the structure, the header accounting and the tone are exactly what
a real run produces.

---

# QA report — ACME-482

- **Target:** http://localhost:3000
- **Run started:** 2026-01-15T09:30:04.118Z
- **Verdict:** `completed`
- **Executors:** 3 in parallel
- **Findings:** 2 reported · 4 dropped as unverified · 1 merged as duplicate

## Findings

### 1. Date filter does not reach the summary cards

**Severity:** Major · **Scenario:** S2

**Steps to reproduce**

1. Open the Orders page as a signed-in user.
2. Note the value of the "Total revenue" card.
3. Set the date filter to "Last 7 days".
4. Wait for the orders table to finish loading.

**Observed:** The table narrows to 12 rows covering the last 7 days, but the three summary cards
above it keep their original values. Reproduced twice from a fresh session; the second run showed
the same split after a hard reload.

**Expected:** The ticket's second acceptance criterion states that the summary cards recompute for
the selected range.

**Verification:** Reproduced from a clean session. Ruled out a caching artefact by changing the
range twice in a row — the table tracked both changes, the cards tracked neither.

**Likely root cause**

`src/pages/Orders/OrdersPage.tsx:118` — the `useEffect` that refetches summary totals lists
`[pageSize]` as its dependency array; `dateRange` is read inside the effect but never declared, so
the summary query is not re-issued when the range changes. The table uses a separate hook that
does declare it, which is why one half of the page updates.

<details><summary>Evidence</summary>

```
GET /api/orders?from=2026-01-08&to=2026-01-15  200
(no request issued to /api/orders/summary after filter change)
```

</details>

---

### 2. Sorting a column clears the active filter without saying so

**Severity:** Minor · **Scenario:** S4

**Steps to reproduce**

1. Open the Orders page and apply the "Pending" status filter.
2. Click the "Amount" column header to sort.

**Observed:** The list re-sorts and silently returns to showing all statuses. The filter chip
still reads "Pending". Reproduced twice.

**Expected:** Sorting should reorder the filtered set, or — if clearing is intended — the chip
should clear with it.

**Verification:** Reproduced from a clean session. Confirmed the chip is stale rather than the
data: the row count matches the unfiltered total while the chip still displays "Pending".

**Likely root cause**

`src/components/DataTable/useTableState.ts:64` — the sort handler resets state with a fresh
object literal rather than spreading the previous state, dropping `activeFilters` on every sort.

---

## How to read this

Each finding was reproduced by a second, independent browser session that was asked to disprove
it. Candidates that could not be reproduced were discarded and are not listed here; they remain in
the run folder alongside the full transcripts.

---

## What the header line is telling you

> **Findings:** 2 reported · 4 dropped as unverified · 1 merged as duplicate

Seven candidates went into verification and two came out. That ratio is the point of the system,
and it is printed on every report on purpose.

The four that were dropped are kept on disk, not deleted:

| Candidate | Verifier's verdict |
| --- | --- |
| "Revenue card truncates large values" | `known-false-positive` — `$89K` is a display abbreviation; the field shows `$89,432` on focus. |
| "Export button is dead" | `not-reproducible` — the button opens a download the first session could not observe. |
| "Order detail shows the wrong customer" | `not-reproducible` — the first session had navigated from a stale list. |
| "Pagination skips page 3" | `not-reproducible` — did not recur in four attempts. |

Any one of those, reported straight through, would have cost a developer an afternoon. The third
one in particular reads as a serious data-integrity bug, and it is a stale list.
