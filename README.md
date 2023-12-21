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
