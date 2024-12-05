# Advent of Flink - Day #3 Working with Arrays


No troubleshooting today. Let's talk about `ARRAY`s. Starting with the basics.
This is how you can create an `ARRAY` manually.

```sql
SELECT ARRAY[1,2,3,4,5];
```
which returns
```
EXPR$0
[1,2,3,4,5]
```
And this is how you reference an element in an array (indices start at 1!): 
```sql
SELECT array_field[1] FROM ((VALUES ARRAY[1,2,3,4,5])) AS T(array_field)
```
which returns
```
EXR$0
1
```
Alright, that's great for testing, but you probably won't create `ARRAY`s a lot manually. Aggregating a field into an 
`ARRAY` in contrast is very common in analytical workloads. For this, CC Flink (like almost all data warehouses) comes 
with the `ARRAY AGG` built-in function

```sql
SELECT 
   window_time,
   user_id, 
   ARRAY_AGG(url) AS urls
 FROM TABLE(TUMBLE(TABLE examples.marketplace.clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end, window_time, user_id;
```
which produces rows like
``` 
window_time               user_id urls
2024-12-03T08:17:59.999Z  3015    [https://www.acme.com/product/sjpfp]
2024-12-03T08:18:59.999Z  3680    [https://www.acme.com/product/jjuqx, https://www.acme.com/product/msbyy]
```
Ok, let's save that query in a view, which we just recently released in CC Flink. 
```sql
CREATE VIEW visited_pages_per_minute AS 
SELECT 
   window_time,
   user_id, 
   ARRAY_AGG(url) AS urls
 FROM TABLE(TUMBLE(TABLE examples.marketplace.clicks, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end, window_time, user_id;
```
Ok, now you've seen how to create arrays both manually and via an aggregation, but what about the opposite? How can I 
expand an array into multiple rows again? That`s done via `CROSS JOIN UNNEST`.
```sql
SELECT v.window_time, v.user_id, u.url FROM visited_pages_per_minute AS v
CROSS JOIN UNNEST(v.urls) AS u(url)
```
which returns (using the output from above)
```
window_time               user_id url
2024-12-03T08:17:59.999Z  3015    https://www.acme.com/product/sjpfp
2024-12-03T08:18:59.999Z  3680    https://www.acme.com/product/jjuqx
2024-12-03T08:18:59.999Z  3680    https://www.acme.com/product/msbyy
```
There is more to be said about arrays, e.g. in the context of nested `ROWs` and JSON strings, but I'll leave that to 
another day.

As always (so far), the examples in here are runnable out of the box on [Confluent Cloud](https://confluent.cloud).
