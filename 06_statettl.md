# Advent of Flink - Day #4 State Time-To-Live

Yesterday, I covered stateful stream processing a bit: you saw that many queries are stateful and that some queries
lead to an ever-growing state, which will over time slow down the query and eventually bring it down. On of these 
queries, was this regular join between `orders` and `customers`. 

```sql
SELECT * FROM `examples`.`marketplace`.`orders` AS o
         INNER JOIN `examples`.`marketplace`.`customers` AS c ON o.customer_id = c.customer_id
```

Now, there are two ways to make the state bounded again. First, we can use a 
[temporal table join](https://docs.confluent.io/cloud/current/flink/reference/queries/joins.html#temporal-joins) 
instead of a regular join.

```sql
SELECT * FROM `examples`.`marketplace`.`orders` AS o
         INNER JOIN `examples`.`marketplace`.`customers` FOR SYSTEM AS OF `$rowtime` AS c ON o.customer_id = c.customer_id
```
This changes the logic of the query. With a temporal table join, every order is enriched with the corresponding customer 
information as of the time when the order happened (`$rowtime`). There is one output row for each `order`. When a row
in `customers` is updated, no rows are emitted. With a regular join on the other hand, every order is joined to the 
most recent customer information *and* when a row in `customers` is updated all matching orders are re-emitted with 
the updated customer information. Therefore, the regular join needs to keep all orders in state, while the temporal 
join doesn't. The temporal join only needs to keep a versioned table of `customers` in state the version of which can 
be pruned as the watermark progressed. 

Temporal Joins are very common and almost always desirable when you join a fact table (`orders`) with one or more 
evolving dimensional tables (`customers`). 

Ok, this was a bit of an excursion. Let's say we actually need the semantics of the regular join. In that case, we can 
configure a state time-to-live. 

```sql
SET 'sql.state-ttl' = '1 day'

SELECT * FROM `examples`.`marketplace`.`orders` AS o
         INNER JOIN `examples`.`marketplace`.`customers` AS c ON o.customer_id = c.customer_id
```

If you configure a state time-to-live, any state that has not been updated for the configured duration becomes 
eligible for cleanup. The actual cleanup happens lazily. So, there in no guarantee that the state is cleaned up after the 
configured time. It's a lower bound. State time-to-live is configured in **processing time**, not event time. This means, 
for example, that state cleanup does not happen more quickly during reprocessing, where event time usually progresses 
much faster than processing time. 

The effect of state expiration varies depending on the operation. In the example above it means that if a row from the LHT
has not been updated for `1 day` any subsequent matching rows from the RHT are not matched and emitted until the 
corresponding row from the LHT is updated again and reappars. If this doesn't happen within yet another day, then that 
RHT row is dropped, too, and the match is lost for good. In contrast, if this was a left, right or full *outer* join, 
state expiration will lead to temporarily (or permanently) partially matched records. 

There are also situations were state time-to-live comes without any downside, but this usually requires additional 
domain knowledge that only the customer can provide. For example: 
* If you know that duplicates are never further apart than 1 day in processing time, there is no downside in configuring
  a state time-to-live of 1 day for a deduplication query. 
* If you know that both sides of a join are never updated later than three months after the row was initially added, 
  it's safe to configure a corresponding state time-to-live. We had this situation e.g. with a customer where the rows 
  the in table described the status of a business process that always concluded within three months.

Finally, there is also the option to configure a different state time-to-live for each side of a join via 
[hints](https://docs.confluent.io/cloud/current/flink/reference/statements/hints.html#state-ttl-hints). 

As always (so far), the examples (except the one with `product_details`) in here are runnable out of the box on 
[Confluent Cloud](https://confluent.cloud).

