# Data quality checks

Source data was loaded into BigQuery and checked before analysis.

| Check | Finding | Action |
|---|---|---|
| Duplicates | 525 duplicate rows in `minuteSleep` | Removed |

## 1. Duplicates

Count rows that repeat another row completely. Run on every table; duplicates were found only in `minuteSleep`.

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(*) - COUNT(DISTINCT TO_JSON_STRING(t)) AS duplicate_rows
FROM `Bellabeat_study.minuteSleep_merged` t
```

**Result:** 525 duplicate rows.
**Action:** removed before analysis.
