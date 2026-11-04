# Data quality checks

Source data was loaded into BigQuery and checked before analysis.

| Check | Finding | Action |
|---|---|---|
| Duplicates | 525 duplicate rows in `minuteSleep` | Removed: created minuteSleep_clean (198,559 → 198,034 rows) |

## 1. Duplicates

Count rows that repeat another row completely. Run on every table; duplicates were found only in `minuteSleep`.

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(*) - COUNT(DISTINCT TO_JSON_STRING(t)) AS duplicate_rows
FROM `Bellabeat_study.minuteSleep_merged` AS t
```
Duplicates removed by saving unique rows into a new table; the raw table is untouched.

```sql
CREATE OR REPLACE TABLE `Bellabeat_study.minuteSleep_clean` AS
SELECT DISTINCT *
FROM `Bellabeat_study.minuteSleep_merged`
```

**Result:** 198,559 rows before, 198,034 after (525 removed).
