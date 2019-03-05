# How are DataFrames converted to RDD? 

The **DataFrame**/**Dataset** is created with a **LogcialPlan**. Roughly, the DataFrame is just a wrapper of the plan. The plans are stored in `Dataset.queryExecution`. The **QueryExecution** holds both the logical and the physical plans. The conversion workflow is: 

## QueryExecution: 

logical: LogicalPlan  (input)\
↓SparkSession.sessionState.analyzer.executeAndCheck\
analyzed: LogicalPlan\
↓SparkSession.sharedState.cacheManager.useCachedData\
withCachedData: LogicalPlan\
↓SparkSession.sessionState.optimizer.executeAndTrack\
optimizedPlan: LogicalPlan\
↓SparkSession.sessionState.planner.plan 
sparkPlan: SparkPlan  (the physical plan)\
↓QueryExecution.prepareForExecution\
executedPlan: SparkPlan\
↓SparkPlan.execute/executeCollect/…\
RDD

## Example: 
### Simple Datas Source
```scala
SparkSession.createDataset(Seq[T]) 
```
* Logical plan:\
   org.apache.spark.sql.catalyst.plans.logical.LocalRelation
* Physical plan:\
  org.apache.spark.sql.execution. LocalTableScanExec 
* RDD:\
  ParallelCollectionRDD 

### Broadcast Join
```scala
DataFrame.join 
```
* Logical plan:\
  org.apache.spark.sql.catalyst.plans.logical.Join 
* Physical plan:\
  BroadcastNestedLoopJoin BuildRight, Inner, true (org.apache.spark.sql.execution.joins. BroadcastNestedLoopJoinExec)\
  :- LocalTableScan [value#1]\
  +- BroadcastExchange IdentityBroadcastMode org.apache.spark.sql.execution.exchange. BroadcastExchangeExec 
  &nbsp;&nbsp;&nbsp;&nbsp;+- LocalTableScan [value#10] 
* RDD:\
  MapPartitionsRDD\
  :- ParallelCollectionRDD\
  +- [broadcast]\
  &nbsp;&nbsp;&nbsp;&nbsp;+- ParallelCollectionRDD

*** TODO ***

Shuffle case