# Advent of Flink - Day #11 User-Defined Event Time Columns

Ok, we are back to time and watermarking. On Day 7 I covered the system column $rowtime. In short, $rowtime contains the Kafka message timestamp and is the default *Rowtime* (also called event time) column, i.e. the column that has a watermark defined on it.
Now, $rowtime is very useful if the Kafka message timestamp contains the actual event timestamp that you want to use. If not, you should use ALTER TABLE to define a watermark on the column that actually is the event/row time column.
Let’s say you have the following table.
```sql
CREATE TABLE clicks_with_click_time AS SELECT 
*, 
EXTRACT(EPOCH FROM TIMESTAMPADD(SECOND,-RAND_INTEGER(30),`$rowtime`)) AS click_time 
FROM `examples`.`marketplace`.`clicks`;
```
and
```sql
DESCRIBE EXTENDED `clicks_with_click_time`;
```
which returns
```
Column Name Data Type	                Nullable	Extras	Comment
click_id	  STRING	                    NOT NULL	 	 
user_id	    INT	                        NOT NULL	 	 
url	        STRING	                    NOT NULL	 	 
user_agent	STRING	                    NOT NULL	 	 
view_time	  INT	                        NOT NULL	 	 
click_time	BIGINT	                    NOT NULL	 	 
$rowtime	  TIMESTAMP_LTZ(3) *ROWTIME*	NOT NULL	METADATA VIRTUAL, WATERMARK AS `SOURCE_WATERMARK`()	SYSTEM
```
`clicks_with_click_time` has a event timestamp called click_time, which is a `BIGINT` and up to 30 seconds out-of-order. Now, in order to use click_time in temporal operations such as `ORDER BY`, `MATCH_RECOGNIZE` or time windows we need to define the watermark on that column. A watermark can only be defined on colunns `TIMESTAMP` or `TIMESTAMP_LTZ`. So, first, we will add a computed column that holds the click_time as a `TIMESTAMP_LTZ`.
```sql
ALTER TABLE `clicks_with_click_time` ADD click_time_ts AS TO_TIMESTAMP_LTZ(click_time, 0);
```
Now, we can modify the watermark for this table and define it on click_time_ts. Of course, we use a bounded out-of-orderness strategy with 30 seconds maximum out-of-orderness.
```sql
ALTER TABLE `clicks_with_click_time` MODIFY WATERMARK FOR `click_time_ts` AS `click_time_ts` - INTERVAL '30' SECONDS;
```
To see if everything looks as expected:
```sql
DESCRIBE EXTENDED `clicks_with_click_time`;
```
which returns
```
Column Name   Data Type	                  Nullable  Extras     	Comment
click_id	    STRING	                    NOT NULL	 	 
user_id	      INT	                        NOT NULL	 	 
url	          STRING	                    NOT NULL	 	 
user_agent	  STRING	                    NOT NULL	 	 
view_time	    INT	                        NOT NULL	 	 
click_time	  BIGINT	                    NOT NULL	 	 
click_time_ts TIMESTAMP_LTZ(3) *ROWTIME*	NULL      AS TO_TIMESTAMP_LTZ(`click_time`, 0), WATERMARK AS `click_time_ts` - INTERVAL '30' SECOND
$rowtime	    TIMESTAMP_LTZ(3)	          NOT NULL  METADATA VIRTUAL	SYSTEM
```
Looks great. `click_time_ts` is marketed as the `*ROWTIME*` column now and we can do temporal operations like
```sql
SELECT * FROM clicks_with_click_time ORDER BY click_time_ts ASC;
```
which would not have been possible without a watermark definition on it. Note, that the $rowtime column still exists and still holds the Kafka Message timestamp. It just doens’t serve as row/event time column anymore.

That’s all for today. As always (so far), the examples in here are runnable out of the box on [Confluent Cloud](https://confluent.cloud).


