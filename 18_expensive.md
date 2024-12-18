# Advent of Flink - Day 18 "Expensive" Operators

On [Day 2](./02_explain.md), I promised to dive deeper in to "expensive operators" or in other words to explain
what to look out for when reading the execution plan comes out of `EXPLAIN`. This day has come. 

Here's the thing: "expensive" can mean many things. Some built-in functions like `JSON_QUERY` or user-defined functions
are very compute-intensive and hence expensive in that sense. Here, I am going to focus on **state**, this means 
[state-intensive](05_state.md) operators, and when it comes to state the most important question is whether the 
state is bounded or unbounded. If it unbounded, meaning growing indefinitely, you need to set a [state time-to-live](.06_statettl.md)
to bound the state. Otherwise, the performance of the statement will sooner or later degrade. The state size also affects
the duration of downtime during recovery and rescaling. 

To this end, we have categorized all operators into 4 categories: 

* **NONE:** stateless
* **LOW:** state is small. state is either used for bookkeeping only or bounded through time attributes. 
* **MEDIUM:** state may be small, large or even unbounded. It often depends on the cardinality of the grouping or join key. For `OVER` aggregation its depends on the cardinality of the partition key and the boundedness of the window. For `MATCH_RECOGNIZE` it depends a lot on the pattern. Needs judgement.
* **HIHG:** state is always unbounded and always needs state time-to-live

The following categorization will become part of the output of `EXPLAIN`. You do not need to learn it by heart.

**NONE**
* StreamPhysicalCalc
* StreamPhysicalCorrelate
* StreamPhysicalDropUpdateBefore
* StreamPhysicalExchange
* StreamPhysicalExpand
* StreamPhysicalUnion
* StreamPhysicalWatermarkAssigner

**LOW**
* StreamPhysicalAsyncCalc
* StreamPhysicalAsyncCorrelate
* StreamPhysicalTableSourceScan
* StreamPhysicalValues
* StreamPhysicalIntervalJoin
* StreamPhysicalTemporalSort
* StreamPhysicalWindowAggregate
* StreamPhysicalWindowDeduplicate
* StreamPhysicalWindowJoin
* StreamPhysicalWindowTableFunction
* StreamPhysicalWindowRank
* StreamPhysicalSink (if no UpsertMaterializer[1] is required)

**MEDIUM**
* StreamPhysicalChangelogNormalize
* StreamPhysicalGlobalGroupAggregate
* StreamPhysicalGlobalWindowAggregate
* StreamPhysicalGroupAggregate
* StreamPhysicalGroupTableAggregate
* StreamPhysicalIncrementalGroupAggregate
* StreamPhysicalLimit
* StreamPhysicalLocalGroupAggregate
* StreamPhysicalLocalWindowAggregate
* StreamPhysicalRank
* StreamPhysicalSortLimit
* StreamPhysicalTemporalJoin
* StreamPhysicalMatch
* StreamPhysicalOverAggregate
* StreamPhysicalJoin (if no table with changelog mode `append` is involved)

**HIGH**
* StreamPhysicalSink (if an UpsertMaterializer[1] is required)
* StreamPhysicalJoin (if at least one table with changelog mode `append` is involved)

This categorization is still being refined as we speak. Also note, that this is a quite coarse-grained categorization. 
Not every operators in the **MEDIUM** category specifically is equally bad and it depends a lot on the specifics of query 
and characteristics of the input streams. This is why, we are prioritizing exposing state size metrics and a query profiling
capability in Confluent Cloud for early 2025. 

You will have noticed that it seems important whether a sink needs an UpsertMaterializer or not. An UpsertMaterializer is 
always needed if a query writes to a table with changelog mode `upsert`, but the query does not produce upsert results where
with upsert keys that correspond to the primary keys of the sink table. Here's an example:

```sql
EXPLAIN CREATE TABLE orders_per_product (
 PRIMARY KEY (product_id) NOT ENFORCED
)
WITH (
  'changelog.mode' = 'upsert'
)
AS SELECT
  product_id,
  COUNT(*) AS cnt
FROM `examples`.`marketplace`.`orders` AS o
GROUP BY product_id
```
leads to 
```sql
...
[4] StreamPhysicalGroupAggregate
Changelog mode: upsert
Upsert key: (product_id)

[5] StreamPhysicalSink
Table: `prod`.`marketplace`.`orders_per_product`
Primary key: (product_id)
Changelog mode: upsert
Upsert key: (product_id)
```
So, this does not require an UpsertMaterializer. Different example:
```sql
EXPLAIN CREATE TABLE orders_products (
  PRIMARY KEY (order_id) NOT ENFORCED
)
WITH (
  'changelog.mode' = 'upsert'
)
AS SELECT
* 
FROM `examples`.`marketplace`.`orders` AS o
INNER JOIN `examples`.`marketplace`.`products` AS p ON p.product_id = o.product_id
```
leads to
```
...
[9] StreamPhysicalJoin
Changelog mode: retract

[10] StreamPhysicalSink
Table: `prod`.`marketplace`.`orders_products`
Primary key: (order_id)
Changelog mode: upsert
```
A retract output needs to become an upsert changelog in the sink. This requires an UpsertMaterializer.

I hope this was a decent 102 into reading Flink SQL query plans and it's all for today. As always (so far), the examples in here are runnable out of the box on [Confluent Cloud](https://confluent.cloud).
