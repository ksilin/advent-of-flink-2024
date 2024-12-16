# Advent of Flink - Day 16 Fun with `OVER´ Aggregations (Event-Driven Alerting)

Today, is going to be a more creative, fun one than the last days.

Let’s say a customer comes to you with the following requirements:

> We would like to block customers in certain situations from placing orders. It’s essentially an anti-fraud or anti-abuse system. For simplicity, let’s say, if a customers places orders worth more than X$ in a certain time interval, or if a customer places more than Y orders per time interval, they should be blocked and a notification should be send. It is important that this happens right away after the order that exceeded the limit was placed - within seconds or so. Now, if the condition is not met anymore when another order is placed, we need another event to unblock the customer.

We will now implement this using Flink SQL.  At the core of our implementation are [`OVER` aggregations](https://docs.confluent.io/cloud/current/flink/reference/queries/over-aggregation.html).

> `OVER` aggregates compute an aggregated value for every input row over a range of ordered rows. In contrast to a GROUP BY aggregate, an OVER aggregate doesn’t reduce the number of result rows to a single row for every group. Instead, an OVER aggregate produces an aggregated value for every input row.

This is good for our requirements, because we don’t want to count orders or sum up prices over a certain time interval (whether tumbling, hopping, session or cumulating), but we want to act on each order immediately.

Let’s say our time interval is 10 seconds. Then, we could start with the following query:
```sql
SELECT 
  order_id,
  customer_id,
  `$rowtime`,
  SUM(price) OVER w AS total_price_ten_secs, 
  COUNT(*) OVER w AS total_orders_ten_secs
FROM `examples`.`marketplace`.`orders`
WINDOW w AS (
  PARTITION BY customer_id
  ORDER BY `$rowtime`
  RANGE BETWEEN INTERVAL '10' SECONDS PRECEDING AND CURRENT ROW
)
```
For every, order we select `order_id`, `customer_id` and `$rowtime` as well as the number (`total_orders_ten_secs`) and total value (`total_price_ten_secs`) 
for the same customer (`PARTITION BY customer_id`) over the last ten seconds (`RANGE BETWEEN INTERVAL '10' SECONDS PRECEDING AND CURRENT ROW`). Run this query 
to understand the output.

Ok, that’s a good start. We could now easily alert on every order that exceeds the limit, but we actually only want to send an event once the customer exceeds the limit for the first time and we want to send another event when the customer drops below the limit again. For this, we’ll do another OVER aggregation over the the existing one so that we can compare total_price_ten_secs  between two consecutive orders.
```sql
WITH orders_ten_secs AS 
( 
SELECT 
  order_id,
  customer_id,
  `$rowtime`,
  SUM(price) OVER w AS total_price_ten_secs, 
  COUNT(*) OVER w AS total_orders_ten_secs
FROM `examples`.`marketplace`.`orders`
WINDOW w AS (
  PARTITION BY customer_id
  ORDER BY `$rowtime`
  RANGE BETWEEN INTERVAL '10' SECONDS PRECEDING AND CURRENT ROW
)
)
SELECT 
  *,
  LAG(total_price_ten_secs, 1) OVER w AS total_price_ten_secs_lag, 
  LAG(total_orders_ten_secs, 1) OVER w AS total_orders_ten_secs_lag
FROM orders_ten_secs
WINDOW w AS (
  PARTITION BY customer_id
  ORDER BY `$rowtime`
)
```
That’s pretty cool. So, now for each order, we not only have the total order value and number of orders for the last ten seconds, but also the same value for when the last order was placed. This allows us to now filter for those orders specifically that pushed the customer above or below the limit. Let’s say the limit for total account value is “$300” and the limit for orders is “5".
```sql
WITH orders_ten_secs AS 
( 
SELECT 
  order_id,
  customer_id,
  `$rowtime`,
  SUM(price) OVER w AS total_price_ten_secs, 
  COUNT(*) OVER w AS total_orders_ten_secs
FROM `examples`.`marketplace`.`orders`
WINDOW w AS (
  PARTITION BY customer_id
  ORDER BY `$rowtime`
  RANGE BETWEEN INTERVAL '10' SECONDS PRECEDING AND CURRENT ROW
)
),
orders_ten_secs_with_lag AS 
(
SELECT 
  *,
  LAG(total_price_ten_secs, 1) OVER w AS total_price_ten_secs_lag, 
  LAG(total_orders_ten_secs, 1) OVER w AS total_orders_ten_secs_lag
FROM orders_ten_secs
WINDOW w AS (
  PARTITION BY customer_id
  ORDER BY `$rowtime`
)
)
SELECT customer_id, 'BLOCK' AS action, `$rowtime` AS updated_at 
FROM orders_ten_secs_with_lag 
WHERE 
(total_price_ten_secs > 300 AND total_price_ten_secs_lag <= 300) OR
(total_orders_ten_secs > 5 AND total_orders_ten_secs_lag <= 5)
UNION ALL 
SELECT customer_id, 'UNBLOCK' AS action, `$rowtime` AS updated_at 
FROM orders_ten_secs_with_lag 
WHERE 
(total_price_ten_secs <= 300 AND total_price_ten_secs_lag > 300) OR
(total_orders_ten_secs <= 5 AND total_orders_ten_secs_lag > 5)
```
Ok, finally, let’s put this into another Kafka topic with a CREATE TABLE AS. I am specifying, customer_id as a primary key, so that Flink interprets every message with the same customer_id as an update to any previous row with the same customer_id.
```sql
CREATE TABLE `customers_status` (
    PRIMARY KEY (customer_id) NOT ENFORCED
)
AS 
WITH orders_ten_secs AS 
( 
SELECT 
  order_id,
  customer_id,
  `$rowtime`,
  SUM(price) OVER w AS total_price_ten_secs, 
  COUNT(*) OVER w AS total_orders_ten_secs
FROM `examples`.`marketplace`.`orders`
WINDOW w AS (
  PARTITION BY customer_id
  ORDER BY `$rowtime`
  RANGE BETWEEN INTERVAL '10' SECONDS PRECEDING AND CURRENT ROW
)
),
orders_ten_secs_with_lag AS 
(
SELECT 
  *,
  LAG(total_price_ten_secs, 1) OVER w AS total_price_ten_secs_lag, 
  LAG(total_orders_ten_secs, 1) OVER w AS total_orders_ten_secs_lag
FROM orders_ten_secs
WINDOW w AS (
  PARTITION BY customer_id
  ORDER BY `$rowtime`
)
)
SELECT customer_id, 'BLOCK' AS action, `$rowtime` AS updated_at 
FROM orders_ten_secs_with_lag 
WHERE 
(total_price_ten_secs > 300 AND total_price_ten_secs_lag <= 300) OR
(total_orders_ten_secs > 5 AND total_orders_ten_secs_lag <= 5)
UNION ALL 
SELECT customer_id, 'UNBLOCK' AS action, `$rowtime` AS updated_at 
FROM orders_ten_secs_with_lag 
WHERE 
(total_price_ten_secs <= 300 AND total_price_ten_secs_lag > 300) OR
(total_orders_ten_secs <= 5 AND total_orders_ten_secs_lag > 5)
```
If we know, run `SELECT * FROM customers_status;` we get a dynamically updating table that holds for every customer (that ever got blocked) its latest action. 
Given the thresholds we picked it actually turns out that most customers are blocked the majority of time with the sample data sources that come with
Confluent Cloud.

That’s it for today. I hope you enjoyed it. I think, its a great example that simple event driven logic can be nicely implemented in SQL. I think, I’ll have more
examples for OVER windows on another day - just because they are so useful. As always (so far), the examples in here are runnable out of the box on [Confluent Cloud](https://confluent.cloud).

