# Advent of Flink - Day 19 `SHOW CREATE TABLE`

A quick one about another default debugging toool: `SHOW CREATE TABLE`. `SHOW CREATE TABLE` is an extremely important command for debugging and support. 
`SHOW CREATE TABLE <table>` shows  the `CREATE TABLE` statement that creates the named table. For example, if we create a table like this. 
```sql
CREATE TABLE products AS SELECT * FROM `examples`.`marketplace.`.`products`;
````
And then run
```sql
SHOW CREATE TABLE products;
```
it will return
```sql
CREATE TABLE `<catalog>`.`<database>`.`products` (
  `product_id` VARCHAR(2147483647) NOT NULL,
  `name` VARCHAR(2147483647),
  `brand` VARCHAR(2147483647),
  `vendor` VARCHAR(2147483647),
  `department` VARCHAR(2147483647),
  CONSTRAINT `PRIMARY` PRIMARY KEY (`product_id`) NOT ENFORCED
)
DISTRIBUTED BY HASH(`product_id`) INTO 4 BUCKETS
WITH (
  'changelog.mode' = 'upsert',
  'connector' = 'confluent',
  'kafka.cleanup-policy' = 'delete',
  'kafka.max-message-size' = '2097164 bytes',
  'kafka.retention.size' = '0 bytes',
  'kafka.retention.time' = '7 d',
  'key.format' = 'avro-registry',
  'scan.bounded.mode' = 'unbounded',
  'scan.startup.mode' = 'earliest-offset',
  'value.format' = 'avro-registry'
)
```

This is valuable in two ways. 

1. It shows you and anyone your sharing this output with all the table options, constraints, distribution keys that were automatically derived when the table was created.
   The most important ones being the primary keys and the changelog mode.
3. You can share the `CREATE TABLE` statement with anyone who needs to reproduce an error that you are seeing. For example, if I was supporting a
   customer and the customer experienced some parser or optimizer error when submitting a statement, I would first ask for the output of
   `SHOW CREATE TABLE` for all involved tables, because that allows me to easily re-create the same tables that the customer is using and then run the
   same query that the customer is having issues with. I don't have any data in input tables yet, but at least the query compiles for me, too, which
   is often already enough but in anycase the first step.

By the way, you can also use `SHOW CREATE TABLE` on the tables in the `examples` catalog, to see how this data is generated. 
```sql
CREATE TABLE `examples`.`marketplace`.`products` (
  `product_id` VARCHAR(2147483647) NOT NULL,
  `name` VARCHAR(2147483647) NOT NULL,
  `brand` VARCHAR(2147483647) NOT NULL,
  `vendor` VARCHAR(2147483647) NOT NULL,
  `department` VARCHAR(2147483647) NOT NULL,
  CONSTRAINT `PK_product_id` PRIMARY KEY (`product_id`) NOT ENFORCED
)
WITH (
  'changelog.mode' = 'upsert',
  'connector' = 'faker',
  'fields.brand.expression' = '#{Commerce.brand}',
  'fields.department.expression' = '#{Commerce.department}',
  'fields.name.expression' = '#{Commerce.productName}',
  'fields.product_id.expression' = '#{Number.numberBetween ''1000'',''1500''}',
  'fields.vendor.expression' = '#{Commerce.vendor}',
  'rows-per-second' = '50'
)
```

That's already it for today. As always (so far), the examples in here are runnable out of the box on [Confluent Cloud](https://confluent.cloud).
   
