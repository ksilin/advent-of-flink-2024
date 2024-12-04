# Advent of Flink - Day #1 `TYPEOF`
As you know, Flink comes with its own [type system](https://docs.confluent.io/cloud/current/flink/reference/datatypes.html). Sometimes you want to be very sure about the type of an expression or column. In those cases, the built-in `TYPEOF` function is your friend.
```sql
SELECT TYPEOF(click_id) FROM `examples`.`marketplace`.`clicks` LIMIT 1;
```
returns
```
STRING NOT NULL
```
In the context of a support ticket, you might ask a customer to run `TYPEOF(event_time)` to confirm if the type of a column they refer to as event_time is `STRING`, `TIMESTAMP` or `TIMESTAMP_LTZ`, which often is hard to disambiguate by just exchanging example values.
`DESCRIBE` or `SHOW CREATE TABLE` also provide this information for a column, but this assumes that you’ve already created a table or view that contains the expression. So, here is another example, where I use this function to make sure that the return type of an expression meets my expectations:
```sql
SELECT
  TYPEOF(window_time), 
 TYPEOF(avg_viewtime)
 FROM (SELECT 
   window_time,
   AVG(view_time) AS avg_viewtime  
 FROM TABLE(TUMBLE(TABLE examples.marketplace.clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
 GROUP BY window_end, window_start, window_time)
 LIMIT 1;
```
returns
```
EXPR$0                    EXPRE$1
TIMESTAMP_LTZ(3) NOT NULL INT NOT NULL
```
Now, I realize I wanted `avg_viewtime` to actually be a `DOUBLE`. So:
```sql
 SELECT
 TYPEOF(window_time), 
 TYPEOF(avg_viewtime)
 FROM (SELECT 
   window_time,
   AVG(CAST(view_time AS DOUBLE)) AS avg_viewtime    
 FROM TABLE(TUMBLE(TABLE examples.marketplace.clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
 GROUP BY window_end, window_start, window_time)
 LIMIT 1;
```
returns
```
EXPR$0                    EXPRE$1
TIMESTAMP_LTZ(3) NOT NULL DOBULE NOT NULL
```

As always all the examples are runnables as is on [Confluent Cloud](https://confluent.cloud). 
