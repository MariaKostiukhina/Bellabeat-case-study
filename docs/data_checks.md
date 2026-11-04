# Data quality checks

Source data was loaded into BigQuery and checked before analysis.

| Check | Finding | Action |
|---|---|---|
| Duplicates | 525 duplicate rows in `minuteSleep` | Removed (saved as `minuteSleep_clean`) |
| Missing values | `Fat` empty in 31 of 33 weight records; no missing values in the other tables | `Fat` not used in analysis |

## 1. Duplicates

Count rows that repeat another row completely. Duplicates were found only in `minuteSleep`.

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(*) - COUNT(DISTINCT TO_JSON_STRING(t)) AS duplicate_rows
FROM `Bellabeat_study.minuteSleep_merged` AS t
```

**Result:** 525 duplicate rows.

Duplicates removed by saving unique rows into a new table; the raw table is untouched.

```sql
CREATE OR REPLACE TABLE `Bellabeat_study.minuteSleep_clean` AS
SELECT DISTINCT *
FROM `Bellabeat_study.minuteSleep_merged`
```

**Result:** 198,559 rows before, 198,034 after (525 removed).

## 2. Missing values

Count empty (`NULL`) values per column. The same query was run on every table; the only gap is in the weight table.

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNTIF(Fat IS NULL) AS missing_fat
FROM `Bellabeat_study.weightLogInfo_merged`
```

**Result:** 33 rows, 31 without `Fat`.
**Action:** the column is not used; filling it would invent data.
