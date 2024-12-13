# Advent of Flink - Day 13 `CASE WHEN`

Before we dive back into time and watemarks tomorrow, let's cover a topic to relex today. Flink SQL comes with a 
[a couple of built-in conditional functions](https://docs.confluent.io/cloud/current/flink/reference/functions/conditional-functions.html). Probably the most 
commonly used one is `CASE WHEN`. It essentially allows you to evaluate conditions and return a value once the first condition is true. Here is a simple example:
```sql
SELECT 
  order_id,
  product_id,
  CASE 
    WHEN price > 30 THEN 'high'
    WHEN price <=30 AND price > 20 THEN 'medium'
    ELSE 'low'
  END AS price_category
FROM `examples`.`marketplace`.`orders`;
```
For what it's worth (I couldn't really think of a particularly useful example.), you can also use `CASE WHEN` in a `WHERE` or `HAVING`-clause like this:
```sql
SELECT 
  *
FROM `examples`.`marketplace`.`orders`
WHERE  
  CASE 
    WHEN price > 30 THEN 'high'
    WHEN price <=30 AND price > 20 THEN 'medium'
    ELSE 'low'
  END IN ('high', 'medium');
````
A common use case for `CASE WHEN` is to manually pivot a table on a fixed number of values, e.g: 
```sql
WITH categorized_orders AS (
SELECT 
  order_id,
  $rowtime,
  CASE 
    WHEN price > 30 THEN 'high'
    WHEN price <=30 AND price > 20 THEN 'medium'
    ELSE 'low'
  END AS price_category
FROM `examples`.`marketplace`.`orders`
)
SELECT 
  window_time,
  SUM(CASE WHEN price_category = 'low' THEN 1 ELSE 0 END) AS count_low, 
  SUM(CASE WHEN price_category = 'medium' THEN 1 ELSE 0 END) AS count_medium, 
  SUM(CASE WHEN price_category = 'high' THEN 1 ELSE 0 END) AS count_high 
FROM TABLE(TUMBLE(TABLE categorized_orders, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end, window_time
```
which returns rows like this
```
window_time              count_low count_medium count_high
2024-12-12T14:59:59.999Z 274       277          1940
2024-12-12T15:00:59.999Z 362       330          2308
```
And that's already it for today. As always (so far), the examples in here are runnable out of the box on [Confluent Cloud](https://confluent.cloud).
