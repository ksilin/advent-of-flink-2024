# Advent of Flink - Day 14 Late Data Handling

As mentioned on Day 7, a watermark of time `t` indicates to the Flink runtime that all messages up to time t have arrived, which allows Flink to take any action that required a certain point in time to be reached, e.g. a time window will be evaluated and the results sent down stream.

A message with a timestamp lower than the current watermark is considered “late” and “late” messages are ignored/discarded by all time-dependent operators. The thing is, you might actually care about every message even if its late, but you also don’t want to apply an outrageously conservative watermark strategy, because that would increase the latency for all results. In Flink DataStream API, there is a concept of “allowed lateness” for this. In Flink SQL, late data handling is more limited right now. It doesn’t have a concept of allowed lateness and there are no metrics or alerts on late messages yet.

Yet, Flink SQL does give you the primitives to detect and handle late messages yourself via our old friend `CURRENT_WATERMARK()`. For example, in the example below we send late messages to a DLQ for further treatment. You an use a STATEMENT SET to perform late data handling and the main logic in the same Flink Job. First, creating the destination tables (Statements Sets currently do not support CREATE TABLE AS but will soon.):

```sql
CREATE TABLE clicks_late
WITH (
 'connector' = 'confluent'
)
LIKE `examples`.`marketplace`.`clicks` (EXCLUDING OPTIONS);
```
and
```sql
CREATE TABLE clicks_counts (
  window_time TIMESTAMP_LTZ(3), 
  cnt BIGINT
);
```
and now we run the Statement Set.
```sql
EXECUTE STATEMENT SET
BEGIN
  INSERT INTO clicks_late SELECT * FROM clicks WHERE `$rowtime` < CURRENT_WATERMARK(`$rowtime`);
  INSERT INTO clicks_counts 
  SELECT window_time, COUNT(*) AS cnt 
  FROM TABLE(TUMBLE(TABLE clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE)) 
  GROUP BY window_start, window_end, window_time;
END
```
And that’s it for today again. As always (so far), the examples in here are runnable out of the box on Confluent Cloud.
