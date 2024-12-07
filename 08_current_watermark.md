# Advent of Flink #8 - The Caveats of `CURRENT_WATERMARK()` 

This is the second bit on watermarks. We continue where we left things off yesterday. As a reminder, we created a table 
called `clicks_4_partitions` and changed the watermak strategy to a bounded out-of-orderness of ten seconds. 

```sql
CREATE TABLE clicks_4_partitions (
  click_id STRING, 
  user_id INT, 
  url STRING,
  user_agent STRING,
  view_time INT, 
  PRIMARY KEY (click_id) NOT ENFORCED
) DISTRIBUTED BY HASH(click_id) INTO 4 BUCKETS
WITH (
    'changelog.mode' = 'append'
)
AS SELECT click_id, user_id, url, user_agent, view_time FROM `examples`.`marketplace`.clicks; 
```
and
```sql
ALTER TABLE `clicks_4_partitions` MODIFY WATERMARK for `$rowtime` AS `$rowtime` - INTERVAL '10' SECONDS
``` 

The output of `DESCRIBE EXTENDED clicks_4_partitions` should show the updated watermark strategy. Alright, let's see 
watermarking in action. 

```sql
SELECT 
   $rowtime, 
   CURRENT_WATERMARK($rowtime) AS wm, 
   TIMESTAMPDIFF($rowtime, CURRENT_WATERMARK(`$rowtime`), SECONDS) AS watermark_lag
FROM clicks;
```

If you've followed along, I guess, you would expect that `watermark_lag` is about 10 seconds in this query. But this
is not at all the case. When I run it the `watermark_lag` fluctuates between anything from 22 seconds to 350 seconds or
even more.

Why is that? There are multiple reasons. First, we need to look at what `CURRENT_WATERMARK()` actually returns. It returns
the current watermark of the **operator**, **before** the current record is processed. The fact that `CURRENT_WATERMARK()` 
takes the time attribute as an argument is very confusing, but the only thing that this function determines based on the
argument is the data type of the timestamp (`TIMESTAMP` vs `TIMESTAMP_LTZ`). Ok, knowing this, we now need to understand
how the operator watermark is determined, and there are two mechanics that are relevant here: 
1. the source watermark is assigned per partition and the operator watermark is the minimum across the watermarks of all 
   the partitions (4 partitions in our case)
2. the operator watermark is only updated about every 200ms in **processing time**

Because of (2.), the watermark will only increase every 200ms, but the event time will increase continuously. 
In consequence, the `watermak_lag` increases continously and drops every 200ms. Now you might wonder: "200ms - 
how bad could that be?". It turns out pretty bad, if you are processing a backlog, this means reading a topic from earliest 
offsets like we do. Because in that case, the `$rowtime` can increase a lot within 200ms of processing time. And that's what 
we are seeing here. So, let's change and read from `latest offsets`, so that event time and processing advance roughly at the 
same speed. This is the behavior that you would expect during stable operations in production. 

```sql 
SET 'sql.tables.scan.startup.mode' = 'latest-offset';

SELECT 
   $rowtime, 
   CURRENT_WATERMARK($rowtime) AS wm, 
   TIMESTAMPDIFF(SECOND, CURRENT_WATERMARK($rowtime), $rowtime) AS watermark_lag
FROM clicks;
```
So, this has already reduced `watermark_lag` to values between 10 and 70 seconds. Why 70 seconds? For this, we need to look into
(1.) Because of (1.), the watermark of the operator only advances based on the watermark of partition from that the source has read 
the least from (in terms of advancing through time). In the meantime, the difference between `watermak_lag` increases, e.g if the 
source operators has read messages with timestamp `100` from all partitions, the watemark is at `100-10 = 90`. Now, the source
continuous to read messages from the first partition up to `$rowtime` equal to `140`. The watermark will not progress, but the 
`watermark_lag` will increase as the `$rowtime` of these messages has increased. Only once, the operator has read 
messages from all partitions up to ``rowtime`` equal to `140`, the watermark will advanced to `130`. With a feature called watermark 
alignment, we prevent that this watermark drift between partitions exceeds 60 seconds by stopping to read from the partitions that are 
ahead in time. So, following that logic, the `watermark_lag` can never exceed 70 seconds. 

To validate this hypothesis, let's do a final experiment with a table with only a single partition: 

```sql
CREATE TABLE clicks_1_partition (
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
Don't forget to change the watermark strategy
```sql
ALTER TABLE `clicks_1_partition` MODIFY WATERMARK for `$rowtime` AS `$rowtime` - INTERVAL '10' SECONDS;
```
and now we see that ``watermark_lg`` lag is indeed pretty stable at 10 seconds. For me, it sometimes goes up to 14 
seconds but that is about it. 

So, what do we learn from this for debugging with `CURRENT_WATERMARK`?
1. `CURRENT_WATERMARK` is kind of unpredictable in reprocessing scenarios, but this is something I heard the engineering team is
   planning to address by aligning the watermark emission frequency to event time instead of processing time, i.e. 
   we would emit more frequent watermarks during re-processing
2. `CURRENT_WATEMRARK` may be delayed up to 60 seconds when reading from multiple partitions
3. `CURRENT_WATERMMAR` can still be extremely helpful to ensure that watermarks are progressing at all. It is often not
   so much the question of whether the watermark is lagging 30 or 180 seconds behind, but wether it is days or years 
   behind. 

That's it for today. Tomorrow, we will return to `SOURCE_WATERMARK()` one more time. 

As always (so far), the examples in here are runnable out of the box on [Confluent Cloud](https://confluent.cloud).
