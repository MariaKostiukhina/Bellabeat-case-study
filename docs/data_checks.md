# Data quality checks

Source data was loaded into BigQuery and checked before analysis.

| Check | Finding | Action |
|---|---|---|
| Duplicates | 525 duplicate rows in `minuteSleep` | Removed (saved as `minuteSleep_clean`) |
| Missing values | `Fat` empty in 31 of 33 weight records; no missing values in the other tables | `Fat` not used in analysis |
| Zero-step days | 61 of 457 days (13%), 14 of 35 users | Kept; excluded from step averages, counted as non-use days |
| Date format | Text with AM/PM in 10 of 11 tables | Converted to `DATETIME` |
| Incomplete days | 224 of 457 days sum to under 1,440 min; in 199 the gap matches minutes asleep (188 within 30 min) | Kept; not treated as missing data |
| Daily vs hourly coverage | Hourly tables cover 1,021 user-days, `dailyActivity` only 457; one user (`4388161847`) is missing from the hourly tables | Usage frequency is counted from the hourly tables; `dailyActivity` used for daily totals only |

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

## 5. Incomplete days

In 224 of 457 days the four activity categories (very active, fairly active, lightly active, sedentary minutes) add up to less than 1,440 minutes. Hypothesis: the missing minutes are time asleep, which these categories do not include.

How many days fall short, and by how much:

```sql
SELECT
  CASE
    WHEN total_min >= 1300 THEN 'almost full day'
    WHEN total_min >= 720 THEN 'more than half day'
    ELSE 'under half day'
  END AS coverage,
  COUNT(*) AS days
FROM (
  SELECT VeryActiveMinutes + FairlyActiveMinutes + LightlyActiveMinutes + SedentaryMinutes AS total_min
  FROM `Bellabeat_study.dailyActivity_merged`
)
WHERE total_min < 1440
GROUP BY coverage
```

**Result:** 17 almost full, 178 more than half a day, 29 under half a day.

Does the gap equal minutes asleep:

```sql
WITH sleep_per_day AS (
  SELECT Id, DATE(date) AS day, COUNT(*) AS sleep_min
  FROM `Bellabeat_study.minuteSleep_clean_parse`
  GROUP BY Id, day
)
SELECT
  COUNT(*) AS days_with_sleep,
  COUNTIF(
    ABS(1440 - (a.VeryActiveMinutes + a.FairlyActiveMinutes + a.LightlyActiveMinutes + a.SedentaryMinutes) - s.sleep_min) <= 30
  ) AS gap_matches_sleep
FROM `Bellabeat_study.dailyActivity_merged` AS a
JOIN sleep_per_day AS s
  ON a.Id = s.Id AND a.ActivityDate = s.day
```

**Result:** 199 days with sleep data; in 188 (94%) the gap is within 30 minutes of minutes asleep.
**Conclusion:** most incomplete days are explained by sleep, not by missing data or an empty battery. The remaining 25 incomplete days have no sleep record and stay unexplained.
**Action:** days kept; no assumption about battery or non-wear is made from them.

## 6. Coverage: daily vs hourly tables

The daily and hourly tables should describe the same days, but the hourly tables contain more than twice as many user-days.

```sql
SELECT
  (SELECT COUNT(*)
   FROM (SELECT DISTINCT Id, DATE(activity_hour) AS day
         FROM `Bellabeat_study.hourlySteps_clean`)) AS hourly_user_days,
  (SELECT COUNT(*)
   FROM `Bellabeat_study.dailyActivity_merged`) AS daily_user_days
```

**Result:** 1,021 user-days in the hourly tables vs 457 in `dailyActivity`.

One user is present in `dailyActivity` but absent from the hourly tables:

```sql
SELECT DISTINCT Id
FROM `Bellabeat_study.dailyActivity_merged`
WHERE Id NOT IN (
  SELECT DISTINCT Id
  FROM `Bellabeat_study.hourlySteps_clean`
)
```

**Result:** user `4388161847`.
**Action:** how often people wear the tracker is measured from the hourly tables, because `dailyActivity` covers less than half of the days they contain. `dailyActivity` is used only for daily totals.

The same pattern was applied to every table; only the table and column names change.

**Result:** every converted table has the same number of rows as its original.
**Action:** analysis uses the converted tables.
