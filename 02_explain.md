# Advent of Flink - Day #2 `EXPLAIN`

`EXPLAIN` is a table stakes troubleshooting and debugging tool for working with Flink SQL on Confluent Cloud. It shows 
you the execution plan for queries you’ve written. Let’s take a look at windowed aggregation from Day 1:
```sql
EXPLAIN SELECT
window_time,
AVG(CAST(view_time AS DOUBLE)) AS avg_viewtime    
FROM TABLE(TUMBLE(TABLE examples.marketplace.clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
GROUP BY window_end, window_start, window_time
```
which returns
```
== Physical Plan ==

StreamPhysicalSink [7]
+- StreamPhysicalCalc [6]
+- StreamPhysicalGlobalWindowAggregate [5]
+- StreamPhysicalExchange [4]
+- StreamPhysicalLocalWindowAggregate [3]
+- StreamPhysicalCalc [2]
+- StreamPhysicalTableSourceScan [1]

== Physical Details ==

[1] StreamPhysicalTableSourceScan
Table: `examples`.`marketplace`.`clicks`

[7] StreamPhysicalSink
Table: Foreground
```
If in contrast, I would have forgotten to group by window_end (because why would I have to group on both start and end 
of a window if one determines the other).
```sql
EXPLAIN SELECT
window_time,
AVG(CAST(view_time AS DOUBLE)) AS avg_viewtime    
FROM TABLE(TUMBLE(TABLE examples.marketplace.clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
GROUP BY window_start, window_time
```
you get
```
== Physical Plan ==

StreamPhysicalSink [8]
+- StreamPhysicalCalc [7]
+- StreamPhysicalGroupAggregate [6]
+- StreamPhysicalExchange [5]
+- StreamPhysicalCalc [4]
+- StreamPhysicalWindowTableFunction [3]
+- StreamPhysicalCalc [2]
+- StreamPhysicalTableSourceScan [1]

== Physical Details ==

[1] StreamPhysicalTableSourceScan
Table: `examples`.`marketplace`.`clicks`

[6] StreamPhysicalGroupAggregate
Changelog mode: retract

[7] StreamPhysicalCalc
Changelog mode: retract

[8] StreamPhysicalSink
Changelog mode: retract
Table: Foreground
```
So, here you could see that because I forgot to group by window_end Flink didn’t recognize this as a [windowed 
aggregation](https://docs.confluent.io/cloud/current/flink/reference/queries/window-aggregation.html) and treated it as
a regular aggregation over a 
[windowed table](https://docs.confluent.io/cloud/current/flink/reference/queries/window-tvf.html), which means 
* ever-growing state size 
* retract instead of append only results
* updates after every record instead of the end of the window
So, quite significant. 
[Interval Joins](https://docs.confluent.io/cloud/current/flink/reference/queries/joins.html#interval-joins) might be 
another example where you’d want to make sure that Flink has recognized your query as such instead of falling back to a 
regular join. I will talk more about what expensive operators to look out for in the output of EXPLAIN on a different 
day.
With `EXPLAIN` you can also see if some intermediate results can be reused, which is particularly useful when working 
with Statement Sets.
```sql
CREATE TABLE `clicks_view_time_1m` (
`window_time` TIMESTAMP(3) WITH LOCAL TIME ZONE NOT NULL,
`avg_viewtime` DOUBLE NOT NULL
)

CREATE TABLE `clicks_view_time_1h` (
`window_time` TIMESTAMP(3) WITH LOCAL TIME ZONE NOT NULL,
`avg_viewtime` DOUBLE NOT NULL
)

EXPLAIN STATEMENT SET
BEGIN
INSERT INTO `clicks_view_time_1m`
SELECT
window_time,
AVG(CAST(view_time AS DOUBLE)) AS avg_viewtime    
FROM TABLE(TUMBLE(TABLE examples.marketplace.clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end, window_time;
INSERT INTO`clicks_view_time_1h`
SELECT
window_time,
AVG(CAST(view_time AS DOUBLE)) AS avg_viewtime    
FROM TABLE(TUMBLE(TABLE examples.marketplace.clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' HOUR))
GROUP BY window_start, window_end, window_time;
END;
```
leads
```
== Physical Plan ==

StreamPhysicalSink [7]
+- StreamPhysicalCalc [6]
+- StreamPhysicalGlobalWindowAggregate [5]
+- StreamPhysicalExchange [4]
+- StreamPhysicalLocalWindowAggregate [3]
+- StreamPhysicalCalc [2]
+- StreamPhysicalTableSourceScan [1]

StreamPhysicalSink [12]
+- StreamPhysicalCalc [11]
+- StreamPhysicalGlobalWindowAggregate [10]
+- StreamPhysicalExchange [9]
+- StreamPhysicalLocalWindowAggregate [8]
+- (reused) [2]

== Physical Details ==

[1] StreamPhysicalTableSourceScan
Table: `examples`.`marketplace`.`clicks`

[7] StreamPhysicalSink
Table: `prod`.`marketplace`.`clicks_view_time_1m`

[12] StreamPhysicalSink
Table: `prod`.`marketplace`.`clicks_view_time_1h`
```
which shows that (only) the source can be reused when aggregating `viewtime` over two different intervals.
And as final remark for this one, please note, that `EXPLAIN SELECT...`  and `EXPLAIN INSERT INTO … SELECT …` might 
yield different plans for the same SELECT query because properties of the destination table can also influence the plan. 
Ah, also we are have - of course - plans to add more information to the output of EXPLAIN.

As always (so far), the examples in here are runnable out of the box on [Confluent Cloud](https://confluent.cloud).
