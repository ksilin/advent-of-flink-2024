# Advent of Flink - Day #15 Nested Rows

We continue with (nested) ROWs. Nested rows are very common, because many schemas are nested to begin with. In addition, Avro Union, Protobuf oneOf, schema references and topics with (Topic)RecordNameStrategy are mapped to tables with nested rows.
So, let’s start by how to create to construct a `ROW`.
```sql
SELECT 
  click_id, 
  view_time, 
  url, 
  CAST((user_id, user_agent) AS ROW<user_id BIGINT, user_agent STRING>) AS user_info,
  `$rowtime`
FROM `examples`.`marketplace`.`clicks`;
```
The `CAST(... AS ROW<user_id BIGINT, user_agent STRING>)` is the only way right now to name the columns of the nested rows. Now, let’s quickly create a view for this.
```sql
CREATE VIEW clicks_with_nested_user_info AS
SELECT 
  click_id, 
  view_time, 
  url, 
  CAST((user_id, user_agent) AS ROW<user_id BIGINT, user_agent STRING>) AS user_info,
  `$rowtime`
FROM `examples`.`marketplace`.`clicks`;
```
The columns of nested rows can be accessed via dot-notation.
```sql
SELECT 
  user_info.user_id, 
  user_info.user_agent,
  user_info.* 
FROM clicks_with_nested_user_info;
```
And another example, because it’s very common. If you have an array of rows, like
```sql
CREATE VIEW page_views_1m AS
SELECT 
  window_time, 
  url,
  ARRAY_AGG(CAST((user_id, view_time, $rowtime) AS ROW<user_id INT, view_time INT, viewed_at TIMESTAMP_LTZ(3)>)) AS page_views
FROM TABLE(TUMBLE(TABLE `examples`.`marketplace`.`clicks`, DESCRIPTOR(`$rowtime`), INTERVAL '1' MINUTE))
GROUP BY window_start, window_end, window_time, url;
```
you can unnest these array like this:
```sql
SELECT window_time, url, T.*
FROM page_views_1m
CROSS JOIN UNNEST(page_views_1m.page_views) AS T(user_id, view_time, viewed_at)
```
Yeah, that’s already it again. Pretty unspectacular, but - hey - it’s Sunday. As always (so far), the examples (except the one with `product_details`) in here are runnable out of the box on 
[Confluent Cloud](https://confluent.cloud).
