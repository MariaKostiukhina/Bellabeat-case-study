# Data quality checks

Source data was loaded into BigQuery and checked before analysis.

| Check | Finding | Action |
|---|---|---|
| Duplicates | 525 duplicate rows in `minuteSleep` | Removed (saved as `minuteSleep_clean`) |
| Missing values | `Fat` empty in 31 of 33 weight records; no missing values in the other tables | `Fat` not used in analysis |
| Zero-step days | 61 of 457 days (13%), 14 of 35 users | Kept; excluded from step averages, counted as non-use days |
| Date format | Text with AM/PM in 10 of 11 tables | Converted to `DATETIME` |

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

## 3. Zero-step days

A person wearing a tracker takes at least some steps in a day, so zero steps is treated as the tracker not being worn.

How many days have zero steps, out of all days in the data:

```sql
SELECT
  COUNT(*) AS total_days,
  COUNTIF(TotalSteps = 0) AS zero_step_days,
  COUNTIF(TotalSteps = 0 AND SedentaryMinutes = 1440) AS zero_and_fully_sedentary
FROM `Bellabeat_study.dailyActivity_merged`
```

**Result:** 457 days, 61 with zero steps (13%), 53 of them fully sedentary.

How many users have such days, out of all users:

```sql
SELECT
  (SELECT COUNT(DISTINCT Id)
   FROM `Bellabeat_study.dailyActivity_merged`) AS total_users,
  (SELECT COUNT(DISTINCT Id)
   FROM `Bellabeat_study.dailyActivity_merged`
   WHERE TotalSteps = 0) AS users_with_zero_step_days
```

**Result:** 14 of 35 users.
**Action:** rows kept. Excluded from step averages, but counted as non-use days when measuring how often people use their devices.


## 4. Date formats

Dates in 10 of the 11 tables were stored as text with AM/PM (e.g. `4/12/2016 12:00:00 AM`):

- hourly: `hourlySteps`, `hourlyCalories`, `hourlyIntensities`
- minute-level: `minuteStepsNarrow`, `minuteCaloriesNarrow`, `minuteIntensitiesNarrow`, `minuteMETsNarrow`, `minuteSleep`
- `weightLogInfo`
- `heartrate_seconds`

They were converted to `DATETIME` and saved in new tables; row counts were checked against the originals. `minuteSleep` was parsed from the de-duplicated table `minuteSleep_clean` and saved as `minuteSleep_clean_parse`. `dailyActivity` already had a `DATE` column and needed no conversion.

```sql
CREATE OR REPLACE TABLE `Bellabeat_study.hourlySteps_clean` AS
SELECT
  Id,
  PARSE_DATETIME('%m/%d/%Y %I:%M:%S %p', ActivityHour) AS activity_hour,
  StepTotal
FROM `Bellabeat_study.hourlySteps_merged`
```

The same pattern was applied to every table; only the table and column names change.

**Result:** every converted table has the same number of rows as its original.
**Action:** analysis uses the converted tables.
