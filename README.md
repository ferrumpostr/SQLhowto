<h2 align="center">SQL HowTo: Totals Across Rows and Columns "in One Action" </h2>

Let's look at a real business problem and see how knowledge of SQL capabilities can make life easier for both the developer and the database.

It usually starts with a business requirement like "here we want to visualize hourly activity with dynamics **by hour and day**" or "we need a report on task **statuses  down by employees with overall totals**" or even "we need a list of documents in progress with their ****total count**** and details ****by performers and clients**".

The essence of all these tasks is roughly the same: we have some raw data in the database, and in the interface, we want to simultaneously get aggregates **in several dimensions**.

Let's try to explore several possible implementations on the database side using the example of the first task with a heat map over a time interval. The goal is to find:

- The count of facts in each "cell" day/hour.
- The count of facts for each day.
- The count of facts for each hour.
- The count of facts for the entire interval.

But first, let's create a table from a million randomly distributed initial "facts."

```
CREATE TABLE timefact AS
  SELECT
    '2023-01-01'::date
      + '1 sec'::interval * (random() * 365 * 86400)::integer ts -- время факта
  FROM
    generate_series(1, 1e6);
-- без индекса - никуда
CREATE INDEX ON timefact(ts);
```

So, let's try to calculate the desired data for the December interval.

Obviously, first, we need to learn how to retrieve data for the "matrix" with coordinates (day, hour):
```
EXPLAIN (ANALYZE, BUFFERS)
SELECT
  ts::date dt
, extract(hour FROM ts) hr
, count(*)
FROM
  timefact
WHERE
  ts BETWEEN '2023-12-01' AND '2023-12-31'
GROUP BY
  1, 2
ORDER BY
  1, 2;
```
Let's take a look at the plan for this query:
<p align="center">
<img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/150/858/fd4/150858fd498cdbf3d3878d4ed61cab3e.png" alt="Alt Text">

Out of 66ms, almost a third was taken up by sorting. In principle, if we can afford to reorder the data on the business logic side, we can dispense with sorting the result:
```
SELECT
  ts::date dt
, extract(hour FROM ts) hr
, count(*)
FROM
  timefact
WHERE
  ts BETWEEN '2023-12-01' AND '2023-12-31'
GROUP BY
  1, 2;
```
This will save us approximately a quarter of the time:
<p align="center">
<img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/5dc/6de/5ec/5dc6de5ec6d91b1fdce8305a85e22a03.png" alt="Alt Text">

  But here comes the interesting part. December is not completed; it's ongoing "right now." Therefore, simply executing four independent queries sequentially with the necessary aggregation on the raw data won't work— the numbers **will diverge**.

This means we need to somehow "freeze" the data, and in PostgreSQL, we can do this in various ways.

## Temporary Table
The first approach involves creating a temporary table with pre-aggregated data:
```
CREATE TEMPORARY TABLE preagg AS
SELECT
  ts::date dt
, extract(hour FROM ts) hr
, count(*)
FROM
  timefact
WHERE
  ts BETWEEN '2023-12-01' AND '2023-12-31'
GROUP BY
  1, 2;
```
Since we still need to "write" the data, executing this query will take approximately **50% longer**. However, from this point onward, everything is simple and fast - each query takes less than 1ms:

```
TABLE preagg;

-- by days
SELECT
  dt
, sum(count)
FROM
  preagg
GROUP BY 1;

--by hours
SELECT
  hr
, sum(count)
FROM
  preagg
GROUP BY 1;

-- "sum"
SELECT
  sum(count)
FROM
  preagg;
```
However, with active use of temporary tables, the system catalog (tables like pg_class, pg_attribute, etc.) may grow, gradually slowing down all queries.

## Multiple Queries in a Transaction

Alternatively, you can consider using a transaction in REPEATABLE READ mode, where each query will "walk" through the original data:

```
BEGIN ISOLATION LEVEL REPEATABLE READ;

-- по "клеткам"
SELECT
  ts::date dt
, extract(hour FROM ts) hr
, count(*)
FROM
  timefact
WHERE
  ts BETWEEN '2023-12-01' AND '2023-12-31'
GROUP BY
  1, 2;

-- по дням
SELECT
  ts::date dt
, count(*)
FROM
  timefact
WHERE
  ts BETWEEN '2023-12-01' AND '2023-12-31'
GROUP BY
  1;

-- по часам
SELECT
  extract(hour FROM ts) hr
, count(*)
FROM
  timefact
WHERE
  ts BETWEEN '2023-12-01' AND '2023-12-31'
GROUP BY
  1;

-- "итого"
SELECT
  count(*)
FROM
  timefact
WHERE
  ts BETWEEN '2023-12-01' AND '2023-12-31';

COMMIT;
```

However, we re-read the original data and calculated the aggregation keys each time, **increasing the query execution time** by approximately 4 times. Not to mention that prolonged transactions (if this report is lengthy) in PostgreSQL can pose issues.

CTE + UNION ALL
So, why not calculate and return all the data in a single query?.

Let's agree on the response format:

- `(dt IS NOT NULL, hr IS NOT NULL)` - "cell"
- `(dt IS NOT NULL, hr IS NULL)` - by days
- `(dt IS NULL, hr IS NOT NULL)` - by hours
- `(dt IS NULL, hr IS NULL)` - "total"

Instead of a temporary table, let's use CTE, and we'll "glue" the results of the queries using UNION ALL:
```
WITH preagg AS (
  SELECT
    ts::date dt
  , extract(hour FROM ts) hr
  , count(*)
  FROM
    timefact
  WHERE
    ts BETWEEN '2023-12-01' AND '2023-12-31'
  GROUP BY
    1, 2
)
  TABLE preagg
UNION ALL
  SELECT
    dt
  , NULL hr
  , sum(count) count
  FROM
    preagg
  GROUP BY 1
UNION ALL
  SELECT
    NULL dt
  , hr
  , sum(count) count
  FROM
    preagg
  GROUP BY 2
UNION ALL
  SELECT
    NULL dt
  , NULL hr
  , sum(count) count
  FROM
    preagg;
```
<p align="center">
<img src="https://habrastorage.org/getpro/habr/upload_files/c4f/adf/7c9/c4fadf7c9340b3d2182719684a0d3821.png" alt="Alt Text">
