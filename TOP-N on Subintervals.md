# SQL HowTo: TOP-N on Subintervals

I periodically encounter similar tasks of the following type: "show the TOP-N positions within each nested interval of a given period."

This could be "the top 5 performers in each semester over the last academic year," or "monthly dynamics of the top 10 best-selling products," or, as in our service for visualizing PostgreSQL plans at explain.tensor.ru, "the top 3 most active countries for each day":

Let's try to create an efficient yet concise query for PostgreSQL that can solve such a task, for example, for the last month.

First, we'll need the primary "facts" table itself, on which we will calculate our "tops." Let's generate a million random records for the "last year" in this table:

```
CREATE TABLE fact4agg AS
  SELECT
    now()::date - (random() * 365)::integer dt      -- дата "факта"
  , chr(ascii('a') + (random() * 26)::integer) code -- код агрегации
  FROM
    generate_series(1, 1e6);
```
Since we'll be making a selection "for a period," an index on the date will definitely come in handy:
```
CREATE INDEX ON fact4agg(dt);
```
## Generating subintervals
At first glance, it might seem sufficient to calculate and output a grouping of "facts" for each day. However, this would be a mistake because some days may have no facts at all.

Certainly, such gaps can be addressed in business logic, but it is not ideally suited for this purpose. Therefore, let's handle this directly in the database using the generate_series function to obtain a series of dates:
```
WITH params(dtb, dte) AS (
  VALUES(now()::date - 30, now()::date)
)
SELECT
  dt::date
FROM
  params
, generate_series(dtb, dte, '1 day') dt;
```
There's just a small nuance here – the generate_series function outputs timestamps, but we need the date type. If you require a different interval, such as a week or a month, simply set a different step: `1 week` or `1 month`.
