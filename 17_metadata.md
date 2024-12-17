# Advent of Flink - Day #17 Reading & Writing Kafka Metadata

Let's quickly talk about how to work with Kafka metadata. For this, we first need a table. 

```sql
CREATE TABLE `clicks` AS SELECT * FROM `examples`.`marketplace`.`clicks`;
```

Now, we can use `ALTER TABLE` to add columns for different pieces of metadata. 

```sql
ALTER TABLE `clicks` ADD (
  `headers` MAP<STRING,STRING> METADATA,
  `leader-epoch`INT METADATA VIRTUAL,
  `offset` BIGINT METADATA VIRTUAL,
  `partition` BIGINT METADATA VIRTUAL,
  `timestamp` TIMESTAMP_LTZ(3) METADATA,
  `timestamp-type` STRING METADATA VIRTUAL,
  `topic` STRING METADATA VIRTUAL
);
```

Read-only columns need to be declared `VIRTUAL`, which exclude them from the schema of the table when the table is written to. As you can see, all 
but `headers` and `timestamp` are read-only, which - if you look at it - makes sense. In addition, virtual columns are by default excluded from a 
`SELECT *` similar to the system column `$rowtime`. The `timetamp` column is equivalent to `$rowtime` - just without a watermark definition. 

Let's take a look: 

```sql
SELECT 
  headers, 
  leader-epoch, 
  offset, 
  partition, 
  timestamp, 
  timestamp-type, 
  topic
FROM clicks LIMIT 10;
```

Ok, so that's about reading. For writing, there is one particular use case, I'd like to point out. Imagine you have a query like the following: 
```sql
SELECT
  window_time,
  COUNT(*) AS events_per_second
FROM TABLE(TUMBLE(TABLE clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' SECOND))
GROUP BY window_time, window_end, window_start
```
Now, you can do just write this into a destination table like this
```sql
CREATE TABLE clicks_per_second (
  window_time TIMESTAMP_LTZ(3),
  events_per_second BIGINT
);

INSERT INTO clicks_per_second 
SELECT
  window_time,
  COUNT(*) AS events_per_second
FROM TABLE(TUMBLE(TABLE clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' SECOND))
GROUP BY window_time, window_end, window_start
````
This works perfectly fine, but the default event time column (`$rowtime`) of `clicks_per_second` will not be very useful like this, because it will 
correspond to roughly the time the record is inserted into the underlying topic, not the logical event time of the message, which is `window_time`. 

Therefore, I'd recommend here to write `window_time` into the Kafka message timestamp, which will be available as default event/row time column 
with `SOURCE_WATERMARK()` as `$rowtime` in `clicks_per_second`: 
```sql
CREATE TABLE clicks_per_second_kts (
  events_per_second BIGINT,
  window_time TIMESTAMP_LTZ(3) METADATA FROM 'timestamp'
);

INSERT INTO clicks_per_second_kts
SELECT
  COUNT(*) AS events_per_second,
  window_time
FROM TABLE(TUMBLE(TABLE clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' SECOND))
GROUP BY window_time, window_end, window_start
```

That's all for today. As always (so far), the examples in here are runnable out of the box on [Confluent Cloud](https://confluent.cloud).
