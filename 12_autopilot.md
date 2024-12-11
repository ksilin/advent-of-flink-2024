# Advent of Flink - Day #12 Why does Autopilot not scale up?

Today, is a shorter one, because I need to go to dinner. On Confluent Cloud, long-running SQL Statements are automatically scaled up and 
down by a component called Autopilot. Autopilot's goal is that all your Statements are keeping up with the changes to their input tables.
In other words, the Kafka consumer lag is zero or decreasing quickly. In addition, Autopilot tries to keep some headroom so that the Statement 
does not need to be scaled up on every little change to the input rate. 

But sometimes, consumer lag ("Messages Behind" in the UI, "pendingRecords" in the metrics API) is increasing and Autopilot is not scaling up. 
Why could this be?

1. So far, `SELECT` Statements always run with a parallelism of 1. Only `CREATE TABELE AS`, `INSERT INTO` and `EXECUTE STATEMENT SET` are considered by Autopilot for scaling.
2. Some operators like a global aggregation (no `GROUP BY`) can not be parallelized and need to run with a prallelism of 1. 
3. Kafka Sources are not scaled beyond the number of partitions. Sometimes, the Kafka Source is chained to other operators, sometimes the
  the whole Statement is a single chained vertex. In that case, all of the operators' parallelism is bound by the number of input partitions.
  We are working on decoupling this.
4. Skew. If the the data is skewed and the Statement is bottlenecked on a single subtask. Additional scale out, is not effective. Autopilot detects this situation and does not
   scale out further.
5. The compute pool is exhausted. Current CFU = Max CFU and Autopilot can not scale out without breaking the customer-defined budget.

(1.) is a limitation we will eventually address as part of a larger rework of interactive `SELECT` queries. (2.) is a pretty fundamental limitation and will probably 
stay around the longest. For (3.), we are considering preventing Kafka sources to be chained to other operators, so that they can scale indepndently. 
For (4.), we have some prototypes that will reduce skew automatically and beyond that we will rely on Query Profiler to surface that problem to the user, 
so that they can rework their query.

Generally, we are working on surfacing the decisions and reasonsing of Autopilot to users via an event log, too, but in the meantime, I thought, it would be useful 
to at least re-iterate what could be happening behind the scenes.
