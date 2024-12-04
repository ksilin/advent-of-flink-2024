# Advent of Flink - Day #4 Views

Let's briefly talk about views in Flink SQL. Take the example from yesterday.

```sql
CREATE VIEW visited_pages_per_minute AS 
SELECT 
   window_time,
   user_id, 
   ARRAY_AGG(url) AS urls
 FROM TABLE(TUMBLE(TABLE examples.marketplace.clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end, window_time, user_id;
```
A view is like a virtual table. It is **not** materialized. It is executed whenever it used in another query. Views
in many ways can be use like a table, e.g. you can describe it
```sql
DESCRIBE visited_pages_per_minute
```
which returns
```
Column Name Data Type	        Nullable	Extras
window_time	        TIMESTAMP_LTZ(3)	    NOT NULL	 
user_id	            INT	                    NOT NULL	 
urls	            ARRAY<STRING NOT NULL>	NOT NULL	 
```
and select from it
```sql
SELECT * FROM visited_pages_per_minute LIMIT 1;
```
which returns 
```
window_time               user_id urls
2024-12-04T13:14:59.999Z  4929    [https://www.acme.com/product/aurlv, https://www.acme.com/product/nppwp]
```
If you are interested in the definition of a view that someone else created just 
```sql
SHOW CREATE VIEW visited_pages_per_minute
```
which returns
```sql
CREATE VIEW visited_pages_per_minute (
  `window_time`,
  `user_id`,
  `urls`
)
AS SELECT `window_time`, `user_id`, ARRAY_AGG(`url`) AS `urls`
FROM TABLE(TUMBLE(TABLE `examples`.`marketplace`.`clicks`, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
GROUP BY `window_start`, `window_end`, `window_time`, `user_id`
```
You can also `ALTER` and `DROP` views. As you can see, views and tables share the same namespace in the catalog. You 
can not have a topic and a table (or topic) with the same name in the same database (or cluster). 
 
Yet, there are also differences between views and tables. Most importantly - as mentioned before - views are not 
materialized: there is neither a Kafka topic nor a schema in SR that backs them. Also, views do not have the `$rowtime` 
system column, do not support metadata columns and you can  generally not define a watermark on them. If you think 
about it, that makes sense, because the watermark is automatically derived by the source tables and the query that 
define the views.

In order to use a view in Confluent Cloud, the principal needs access to both the view itself *and* all the 
tables and schemas that the view uses in its definition. 

When to use a view? The main use case for a view is to encapsulate some logic so that it can be reused in many different
queries by different users. This improves a layer of abstraction and improves maintainability and ease of use. If an 
expensive  view is used in many places, it may make more sense to use a materialized views (a.ka. a table in 
CC Flink) to avoid redundant processing in all the queries that use the view.

As always (so far), the examples in here are runnable out of the box on CC Flink and questions, remarks, corrections and 
discussion are very welcome in the thread.**
