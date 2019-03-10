# Appendix A Minor concepts
These are concepts you may encounter when reading the code. They may not be critical to understand the main logic. The explanations mostly come from the Javadoc in the source code. 

## CallSite
(For information display) When called inside a class in the spark package, returns the name of the user code class (outside the spark package) that called into Spark, as well as which Spark method they called. This is used, for example, to tell users where in their code each RDD got created. (see `Util.getCallSite`)

##BarrierStage
(Spark 2.4) see `RDD.isBarrier` and [SPARK-24374](spark 2.4 https://issues.apache.org/jira/browse/SPARK-24374):
```
Whether the RDD is in a barrier stage. Spark must launch all the tasks at the same time for a barrier stage. An RDD is in a barrier stage, if at least one of its parent RDD(s), or itself, are mapped from an [[RDDBarrier]]. This function always returns false for a [[ShuffledRDD]], since a [[ShuffledRDD]] indicates start of a new stage. A [[MapPartitionsRDD]] can be transformed from an [[RDDBarrier]], under that case the [[MapPartitionsRDD]] shall be marked as barrier. 
```

## SparkEnv
Holds all the runtime environment objects for a running Spark instance (either master or worker), including the serializer, RpcEnv, block manager, map output tracker, etc. Currently Spark code finds the SparkEnv through a global variable, so all the threads can access the same SparkEnv. It can be accessed by `SparkEnv.get`. 

## OutputCommitCoordinator
 For *speculative execution*.  This class was introduced in [SPARK-4879](https://issues.apache.org/jira/browse/SPARK-4879); see that JIRA issue (and the associated pull requests) for an extensive design discussion
 ```
 Authority that decides whether tasks can commit output to HDFS. Uses a "first committer wins" policy. OutputCommitCoordinator is instantiated in both the drivers and executors. On executors, it is configured with a reference to the driver's OutputCommitCoordinatorEndpoint, so requests to commit output will be forwarded to the driver's OutputCommitCoordinator.. 
```

## BlacklistTracker
```
BlacklistTracker is designed to track problematic executors and nodes.  It supports blacklisting executors and nodes across an entire application (with a periodic expiry).  TaskSetManagers add additional blacklisting of executors and nodes for individual tasks and stages which works in concert with the blacklisting here. 
```