# Advent of Flink - Day #7 `$rowtime` & `SOURCE_WATERMARK()` 

`$rowtime` is a system column that is available for all tables in Confluent Cloud. It corresponds to the Kafka message 
timestamp. The column does not appear in the output of a regular `DESCRIBE`, but its listed when you 
`DESCRIBE EXTENDED`. Similarly, it is not returned via a `SELECT *`, but it needs to be selected explicitly.

```sql
DESCRIBE EXTENDED `examples`.`marketplace`.`clicks`;
```
which returns
````
Column Name Data Type	                  Nullable	Extras	                                           Comment
click_id	  STRING	                    NOT NULL	 	 
user_id	    INT	                        NOT NULL
url	        STRING	                    NOT NULL	 	 
user_agent	STRING	                    NOT NULL	 	 
view_time	  INT	                        NOT NULL	 	 
$rowtime	  TIMESTAMP_LTZ(3) *ROWTIME*	NOT NULL	METADATA VIRTUAL, WATERMARK AS `SOURCE_WATERMARK`()	SYSTEM
````
`$rowtime` is the default `*ROWTIME*` column for every table in CC Flink, which means the column that has the watermark 
defined on it. At any point in time, there can only be one `*ROWTIME*` column per table. As a reminder: A watermark in 
Flink is a marker that keeps track of time as data is processed. A watermark of time `t` indicates to the Flink runtime 
that all rows with a timestamp lower than `t` have already arrived. Messages with a timestamp higher than the current watermark, 
are considered "late" and are usually dropped from time-based operations like windows or match_recognize. Therefore, for example, 
Flink evaluates time windows after it has seen a watermark higher than the `window_end`, because that indicates that all rows that 
should fall into the window have arrived.

The default watermark strategy applied to the `$rowtime` column is `SOURCE_WATERMARK()`, i.e. the watermark defined by
the source. In order to understand the `SOURCE_WATERMARK()` strategy, we first need to talk about the most common watermarking 
strategy in Apache Flink: bounded out-of-orderness. This means that the timestamp of any message is at most `maximum-out-of-orderness` 
lower than the highest timestamp seen so far. To play around with different watermarking strategies, we need can not use the tables in 
`examples.markeplace`, because those table definition are immutable. So, let's create our own `clicks` table from it. 

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
Assuming a maximum out of orderness of 10 seconds, we could alter the `$rowtime` 
watermark strategy to bounded out of orderness via

```sql
ALTER TABLE `clicks_4_partitions` MODIFY WATERMARK for `$rowtime` AS `$rowtime` - INTERVAL '10' SECONDS
```

The watermark strategy allows us to trade off between completeness and latency. The higher the maximum out-of-orderness, the fewer "late"
message we will have and the more complete our results will be. A more aggressive watermark strategy means more late data, but lower latency results.

Now, the idea of the `SOURCE_WATERMARK()` is to automatically choose and adjust the maximum out-of-orderness over time based on the 
out-of-ordness that it observes in the stream. Quoting [documentation](https://docs.confluent.io/cloud/current/flink/reference/functions/datetime-functions.html#flink-sql-source-watermark-function):

*Watermarks are assigned per |ak| partition in the source operator. They
are based on a moving histogram of observed out-of-orderness in the table,
i.e. the difference between the current event time of an event and the maximum
event time seen so far.*

*The watermark is then assigned as the maximum event time seen so far minus
the 95% quantile of observed out-of-orderness. In other words: the default
watermark strategy aims to assign watermarks so that at most 5% of messages
are "late", i.e. arrive after the watermark.*

*The minimum out-of-orderness is 50ms. The maximum out-of-orderness is 7 days.*

*The algorithm always considers the out-of-orderness of the last 5000 events
per partition. During warmup - before the algorithm has seen 1000 messages
(per partition) it applies an additional safety margin to the observed
out-of-orderness. The safety margin depends on the number of messages seen so
far.*

================================== =================================
Number of Message                  Safety Margin
================================== =================================
1 - 250                            7 days
251 - 500                          30s
501 - 750                          10s
751 - 1000                         1s
================================== =================================

*In effect, the algorithm doesn't provide a usable watermark before it has seen
250 records per partition.*

That's it for today. Tomorrow, we will continue to look into `SOURCE_WATERMARK()` and try to use `CURRENT_WATERMARK()` function 
to observe the watermark strategy as it adjust the maximum out-of-orderness over time.

As always (so far), the examples in here are runnable out of the box on [Confluent Cloud](https://confluent.cloud).

