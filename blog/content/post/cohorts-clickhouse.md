---
title: "Cohorts in Redash with Clickhouse"
date: 2019-04-23T09:19:00+01:00
draft: false
---

Cohorts are a common way to display and track the retention of a service (buyers, active users, whatever_you_need). As I said [before](redpush) in my team we heavily use [Redash](https://github.com/getredash/redash), and we use it together with [Clickhouse](https://clickhouse.yandex/). To get the cohorts calculated, you need some SQL tricks. I couldn't find much on the Internet on how to do it, so here is our contribution.

{{< figure src="/cohort-example.png" alt="Cohort example">}}

Clickhouse is really powerful but it has its own special flavor of SQL. It is becoming closer to the standard with recent versions. We are not running the latest version, so here it is how we managed to get them running. Once you have the basics, you can extend it to any other use case.

Lets start defining an example database schema that we are going to use during this example. Imagine we have a service where users need to register. When that happens we have an entry in the `Accounts` table. To simplify the example, imagine that each time there is some user interaction with the service we have an entry in the `Actions` table.

{{< figure src="/cohort-db-schema.png" alt="Diagram DB schema">}}

We are going to build a cohort to analyze how users remain active in the following weeks since their registration. In this example we use a period of two months (starting from 1st day of the month). Just counting one interaction might not be enough to track if a user is active or not. But we leave that for another discussion. More logic can be added later if needed.

We start from the most inner part of the SQL query:

```sql
    (SELECT AccountID, /* accounts that are active, the non active are not excluded */
              RegisterDate
       FROM
         (SELECT ID as AccountID, toMonday(EventDate) AS RegisterDate /* newReg accounts since 2 months ago */
          FROM Accounts
          WHERE EventDate >= toStartOfMonth(today() - INTERVAL 2 MONTH) /* start of 2 months ago */
       ALL LEFT  JOIN /* all left, as if account was not active we want to keep it */
         (SELECT toMonday(EventDate) AS ActionTime, /* accounts active since two months ago, grouped per week */
                 AccountID
          FROM Actions
          WHERE EventDate >= toStartOfMonth(today() - INTERVAL 2 MONTH) /*  start of 2 months ago */
          GROUP BY ActionTime, AccountID)  /* we want only one entry per week and accountID */
        USING AccountID)
    GROUP BY AccountID, RegisterDate
    )
```

With that query we should have a list of entries for the period of the Accounts registered, with the registration date, and the weeks they were active. Comments added to the query to clarify what it is doing. We get all accounts registered in the period we are interested and join it (without loosing any if not present on the right) with a weekly summary of the activity of those accounts. Here is where you could add more complex logic if needed for your use case (not just one interaction needed).

Once we have that, we apply some Clickhouse's array magic to start getting the data we need:

```sql
    SELECT AccountID, /* calculate if each of the weeks after each account was still active */
           groupArray(toUInt64(RegisterDate)) AS ActiveTimes,
           toUInt64(RegisterDate) as RegisterDateStamp, /* RegisterDate in timestamp format to compare in the filter phase */
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp AND  activeTime < RegisterDateStamp + 7, ActiveTimes)[1] as week1,
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 7  AND  activeTime < RegisterDateStamp + 14, ActiveTimes)[1] as week2,
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 14  AND  activeTime < RegisterDateStamp + 21, ActiveTimes)[1] as week3,
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 21  AND  activeTime < RegisterDateStamp + 28, ActiveTimes)[1] as week4,
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 28  AND  activeTime < RegisterDateStamp + 35, ActiveTimes)[1] as week5,
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 35  AND  activeTime < RegisterDateStamp + 42, ActiveTimes)[1] as week6,
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 42  AND  activeTime < RegisterDateStamp + 49, ActiveTimes)[1] as week7,
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 49  AND  activeTime < RegisterDateStamp + 56, ActiveTimes)[1] as week8,
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 56  AND  activeTime < RegisterDateStamp + 63, ActiveTimes)[1] as week9,
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 63  AND  activeTime < RegisterDateStamp + 70, ActiveTimes)[1] as week10,
           arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 77  AND  activeTime < RegisterDateStamp + 77, ActiveTimes)[1] as week11
    FROM
        (SELECT AccountID, /* accounts that are active, the non active are not excluded */
                RegisterDate
        FROM
            (SELECT ID as AccountID, toMonday(EventDate) AS RegisterDate /* newReg accounts since 2 months ago */
            FROM Accounts
            WHERE EventDate >= toStartOfMonth(today() - INTERVAL 2 MONTH) /* start of 2 months ago */
        ALL LEFT  JOIN /* all left, as if account was not active we want to keep it */
            (SELECT toMonday(EventDate) AS ActionTime, /* accounts active since two months ago, grouped per week */
                    AccountID
            FROM Actions
            WHERE EventDate >= toStartOfMonth(today() - INTERVAL 2 MONTH) /*  start of 2 months ago */
            GROUP BY ActionTime, AccountID)  /* we want only one entry per week and accountID */
            USING AccountID)
    GROUP BY AccountID, RegisterDate
```

The first trick is to convert the dates to "`DateStamps`" (days since epoch) using `toUInt64`. It will make the checks much easier. The second trick is to put all the `ActiveTime` into an array. For that we use the [groupArray](https://clickhouse.yandex/docs/en/query_language/agg_functions/reference/#grouparray-x-grouparray-max-size-x) function. Which will append all the weekly active times per user (we are grouping per AccoundID and Register date).

Last trick in this query is to use the lambda functions and a [filter](https://clickhouse.yandex/docs/en/query_language/functions/higher_order_functions/#arrayfilter-func-arr1) to get new columns if the user was active or not on each of the weeks. You need to be careful with all the copy pasting here.

As a result you should have something like

|AccountID	| ActiveTimes| RegisterDateStamp|	week1|	week2|	week3|	week4|	week5|	week6|	week7|	week8|	week9|	week10|	week11|
| ------ | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| 1234 | ["17987","17973","17980"] | 17,973 | 17,973 | 17,980 | 17,987 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| 4567 | ["17966","17959","17987","17980","18008","17952","18001","17973","17994"] | 17,952 | 17,952 | 17,959 | 17,966 | 17,973 | 17,980 | 17,987 | 17,994 | 18,001 | 18,008 | 0 | 0 |

Now comes the two final outer queries with the tricks to get the result you need for Redash to be able to display the cohort. This is the final query:

```sql
SELECT RegistrationWeek as date, Registrations as Total, Value, WeekNumber FROM 
    (SELECT toDate(RegisterDateStamp) as RegistrationWeek, /* convert back to date */
            count() as Registrations,
            sum(week1 != 0) as week1,
            sum(week2 != 0) as week2,
            sum(week3 != 0) as week3,
            sum(week4 != 0) as week4,
            sum(week5 != 0) as week5,
            sum(week6 != 0) as week6,
            sum(week7 != 0) as week7,
            sum(week8 != 0) as week8,
            sum(week9 != 0) as week9,
            sum(week10 != 0) as week10,
            sum(week11 != 0) as week11,
            [week1, week2 ,week3 ,week4 ,week5,week6,week7,week8,week9,week10,week11] as perWeekArray
    FROM (
        SELECT AccountID, /* calculate if each of the weeks after each account was still active */
            groupArray(toUInt64(RegisterDate)) AS ActiveTimes,
            toUInt64(RegisterDate) as RegisterDateStamp, /* RegisterDate in timestamp format to compare in the filter phase */
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp AND  activeTime < RegisterDateStamp + 7, ActiveTimes)[1] as week1,
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 7  AND  activeTime < RegisterDateStamp + 14, ActiveTimes)[1] as week2,
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 14  AND  activeTime < RegisterDateStamp + 21, ActiveTimes)[1] as week3,
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 21  AND  activeTime < RegisterDateStamp + 28, ActiveTimes)[1] as week4,
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 28  AND  activeTime < RegisterDateStamp + 35, ActiveTimes)[1] as week5,
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 35  AND  activeTime < RegisterDateStamp + 42, ActiveTimes)[1] as week6,
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 42  AND  activeTime < RegisterDateStamp + 49, ActiveTimes)[1] as week7,
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 49  AND  activeTime < RegisterDateStamp + 56, ActiveTimes)[1] as week8,
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 56  AND  activeTime < RegisterDateStamp + 63, ActiveTimes)[1] as week9,
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 63  AND  activeTime < RegisterDateStamp + 70, ActiveTimes)[1] as week10,
            arrayFilter(activeTime -> activeTime >= RegisterDateStamp + 77  AND  activeTime < RegisterDateStamp + 77, ActiveTimes)[1] as week11
        FROM
            (SELECT AccountID, /* accounts that are active, the non active are not excluded */
                    RegisterDate
            FROM
                (SELECT ID as AccountID, toMonday(EventDate) AS RegisterDate /* newReg accounts since 2 months ago */
                FROM Accounts
                WHERE EventDate >= toStartOfMonth(today() - INTERVAL 2 MONTH) /* start of 2 months ago */
            ALL LEFT  JOIN /* all left, as if account was not active we want to keep it */
                (SELECT toMonday(EventDate) AS ActionTime, /* accounts active since two months ago, grouped per week */
                        AccountID
                FROM Actions
                WHERE EventDate >= toStartOfMonth(today() - INTERVAL 2 MONTH) /*  start of 2 months ago */
                GROUP BY ActionTime, AccountID)  /* we want only one entry per week and accountID */
                USING AccountID)
        GROUP BY AccountID, RegisterDate
    ) GROUP by RegistrationWeek /* final grouping per registration to count each of the cumulatives of users per week */
) ARRAY JOIN perWeekArray as Value , arrayEnumerate(perWeekArray) AS WeekNumber
```

Out of those two new queries, the first one is:

```sql
        SELECT toDate(RegisterDateStamp) as RegistrationWeek, /* convert back to Date */
            count() as Registrations,
            sum(week1 != 0) as week1,
            sum(week2 != 0) as week2,
            sum(week3 != 0) as week3,
            sum(week4 != 0) as week4,
            sum(week5 != 0) as week5,
            sum(week6 != 0) as week6,
            sum(week7 != 0) as week7,
            sum(week8 != 0) as week8,
            sum(week9 != 0) as week9,
            sum(week10 != 0) as week10,
            sum(week11 != 0) as week11,
            [week1, week2 ,week3 ,week4 ,week5,week6,week7,week8,week9,week10,week11] as perWeekArray
        FROM (
            …………………
        ) GROUP by RegistrationWeek /* final grouping per registration to count each of the cumulatives of users per week */
```

Which goes through the results of the previous query, grouping per week. It counts how many registrations were in each week, and also counts how many users were active on each of the following weeks. One nice trick is to use the `sum()` function with a condition instead of a column. So if that condition is _true_ it is a `1` and if not it is a `0`. Summing those in the end means that it is counting the amount of entries that match that condition. Once we have all the counts we put them again in an array. This is just the final trick, to produce the data in the format Redash expects it.

And the last query:

```sql
SELECT RegistrationWeek as date, Registrations as Total, Value, WeekNumber FROM 
    ………………
) ARRAY JOIN perWeekArray as Value , arrayEnumerate(perWeekArray) AS WeekNumber
```

We use Clickhouse powerful arrays and `array join`. Redash to display a cohort needs 3 things:

- Bucket. Y-axis, in our case weeks.
- Population size of the bucket (in our case the amount of users registered in a week).
- Each of the values of each of the stages. In two columns, _stage, value_. To be able to create those two columns, stage and size of that stage is why we use the `ARRAY JOIN`, and we pass two values to it: The array of counts of each stage we calculated in the previous step, and then [arrayEnumerate](https://clickhouse.yandex/docs/en/query_language/functions/array_functions/#array_functions-arrayenumerate) which returns the index to the position in the array. Resulting in something like:

| date | Total | Value | WeekNumber |
| ------ | ------ | ------ |:------:|
| 28/01/19 | 2,589 | 2,096 | 1 |
| 28/01/19 | 2,589 | 1,938 | 2 |

With that result we just need to point Redash to use the appropriate columns for each of the values and you should have your cohort as in the picture at the top. It is important to note that Redash has also the opportunity to do date grouping by itself. If the granularity returned by the queries is in days, it can do the cohort afterwards in weeks or months. How you do depends on your use case, flexibility you need and how much data you have. As Clickhouse is much faster doing the groupings that Redash (and saves the data transfer).

I hope it helps!!