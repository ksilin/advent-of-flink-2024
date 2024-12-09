# Advent of Flink - Day #9 A final look at `SOURCE_WATERMARK()` 

Today, is going to be a short one and we are wrapping up the coverage of `CURRENT_WATERMARK()` and `SOURCE_WATERMARK()`. 

Let's see if we can use `CURRENT_WATERMARK()` to observe the behavior or `SOURCE_WATERMARK()`. As we've seen yesterday, 
we better use a table with a single partition for this, because otherwise everything will be blurred by watermark skew
between the partitions. 

```sql
CREATE TABLE clicks_1_partition_source_watermark (
  click_id STRING, 
  user_id INT, 
  url STRING,
  user_agent STRING,
  view_time INT, 
  PRIMARY KEY (click_id) NOT ENFORCED
) DISTRIBUTED BY HASH(click_id) INTO 1 BUCKETS
WITH (
    'changelog.mode' = 'append'
)
AS SELECT click_id, user_id, url, user_agent, view_time FROM `examples`.`marketplace`.clicks; 
```
Alternatively, you can also re-use the same table as yesterday. Simply, use 
```sql
ALTER TABLE `clicks_1_partition` MODIFY WATERMARK FOR `$rowtime` AS SOURCE_WATERMARK();
```
to change the watermark back to `SOURCE_WATERMARK()`. Now we can run the same query as yesterday but we limit the output to 
5000 rows, which is the amount of records the `SOURCE_WATERMARK()` algorithm takes into consideration.

```sql
SET 'sql.tables.scan.startup.mode' = 'latest-offset';

SELECT 
   $rowtime, 
   CURRENT_WATERMARK($rowtime) AS wm, 
   TIMESTAMPDIFF(SECOND, CURRENT_WATERMARK($rowtime), $rowtime) AS watermark_lag
FROM clicks_1_partition_source_watermark LIMIT 5000;
```
From the output you can see the following: 

* the watermark *before* the first row is `NULL`, which makes sense: there is no information to base this on. 
* You can then see the safety margin of the algorithm drop from 7 days to 30 seconds to 10 seconds to 1 seconds after 250, 500, 750 and 100 messages as described in [Day 7](./07_rowtime.md). 
* You can see that the watermark lag is reduced up to the 0 seconds (pretty much the minimum of the algorithm), because the source data is perfectly in order and the algorithm recognizes that.

And that's already it for today. As always (so far), the examples in here are runnable out of the box on Confluent Cloud.
