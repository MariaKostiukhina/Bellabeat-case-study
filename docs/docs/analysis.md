# Analysis

Three questions are answered here; the queries use the cleaned tables (see `data_checks.md`).

## Finding 1: which metrics do people track?

How many of the 35 users appear in each table:

```sql
SELECT
  (SELECT COUNT(DISTINCT Id) FROM `Bellabeat_study.dailyActivity_merged`) AS activity,
  (SELECT COUNT(DISTINCT Id) FROM `Bellabeat_study.hourlySteps_clean`) AS hourly_steps,
  (SELECT COUNT(DISTINCT Id) FROM `Bellabeat_study.minuteSleep_clean_parse`) AS sleep,
  (SELECT COUNT(DISTINCT Id) FROM `Bellabeat_study.heartrate_seconds_clean`) AS heart_rate,
  (SELECT COUNT(DISTINCT Id) FROM `Bellabeat_study.weightLogInfo_clean`) AS weight
```

**Result:**

| Metric | Users | Share of 35 |
|---|---|---|
| Activity | 35 | 100% |
| Hourly steps / calories / intensity | 34 | 97% |
| Sleep | 23 | 66% |
| Heart rate | 14 | 40% |
| Weight | 11 | 31% |

**Takeaway:** all 35 users track activity, but only two thirds track sleep, 40% heart rate and under a third weight. Activity is the core feature people use; the others reach far fewer users.

## Finding 2: how regularly do people wear the tracker?

_(to be added)_

## Finding 3: when does activity drop?

_(to be added)_
