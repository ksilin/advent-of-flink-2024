# Advent of Flink - Day #5 Stateful and Stateless Queries

Today, we are back with an operational topic: state (today) and state time-to-live (tomorrow). Many Flink SQL queries 
need to maintain what we call "state" in stream processing. State is always needed if the output of processing a row is 
not only determined by that row itself, but also depends on the rows, which have previously been processed.

The following operations lead to stateful queries: 
* any JOINs (except `CROSS JOIN UNNEST` & `LATERAL`)
* `GROUP BY` windowed or non-windowed aggregation
* `OVER` aggregation
* `MATCH_RECOGNIZE`

On the other hand, queries that only use
* `INSERT INTO`, `FROM` (reading and writing to Kafka)
* `WHERE` (filters)
* `CROSS JOIN UNNEST` & `LATERAL`
* `SELECT` (projection)
*  scalar and table functions
are stateless.

Operationally, it is very important to understand whether the state that needs to be maintained for a Statement is 
growing indefinitely, or whether it is bounded (or growing so slowly that it is effectively bounded.). Whether the 
state of a query the cardinality of the join/group by-columns.

Let's take a look at some examples:

```sql
SELECT customer_id, COUNT(*) AS num_orders 
FROM `examples`.`marketplace`.`orders`
GROUP BY customer_id
```
This statement needs to maintain on `BIGINT` per customer. So, if the number of customers is growing quickly and 
indefinitely, the state will grow as well. In practice, this will often be ok, because number of customers is not 
growing very quickly and the state per customer is small.

```sql
SELECT * FROM `examples`.`marketplace`.`orders` AS o
         INNER JOIN `examples`.`marketplace`.`customers` AS c ON o.customer_id = o.customer_id
```
This join needs to materialize both tables `orders` and `customers` fully in state. This is because if a row in the 
left-hand table (LHT) is updated, the operator needs to emit an updated match for with all matching rows in the 
right-hand table (RHT). The cardinality of `customers` is probably pseudo-bounded as discussed above, but the 
cardinality of `orders` is - if we have a successful business - unbounded and growing much more quickly than the number 
of customers.

```sql
SELECT *  FROM `examples`.`marketplace`.`products` AS p
          INNER JOIN `examples`.`marketplace`.`product_details` AS pd ON pd.product_id = p.product_id
```
The state of this join is probably manageable. Again, we need to keep both tables `products` and `product_details` in 
state, but this time both tables are bounded or pseudo-bounded so that state is only growing so slowly, that it is 
irrelevant in practice.

And, as a final example: 

```sql 
SELECT 
    customer_id,
    ARRAY_AGG(`order_id`) AS order_ids
FROM `examples`.`marketplace`.`orders`
GROUP BY customer_id
```
Even though, the cardinality of `customers` is manageable, this time the state per customer is growing quickly as we 
store the id of every order that this customer has ever placed.

In Confluent Cloud, we currently do not expose metrics on state size. This is planned for Q1/25 as part of the Query 
Profiler. Until then, it is even more important to develop some intuition and experience on what type of queries are 
potentially unsustainable.

Tomorrow, we will look into the main strategy to mitigate ever-growing state size: state time-to-live and the trade-offs
it involves.

As always (so far), the examples (except the one with `product_details`) in here are runnable out of the box on 
[Confluent Cloud](https://confluent.cloud).

