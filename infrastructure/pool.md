 # Task Scheduler Pool
 The `TaskScheduler` is initialized with a root **Pool** and a **SchedulableBuilder**, which can be either `FIFOSchedulableBuilder` or `FairSchedulableBuilder`. The scheduling mode is decided by the configuration `spark.scheduler.mode` in the `SparkContext`. It’s **FIFO** by default.

## Schedulable Tree ##
Notice the Pool can be **nested**. It’s a tree-like structure. The tree nodes are **Schedulable**. There are two types of Schedulable: **Pool** itself and **TaskSetManager**. The Pool is the branch node, with its children in the field `schedulableQueue` and the TaskSetManager is the leaf node. It contain a `TaskSet` for scheduling. 

There are 5 fields in the Schedulable for scheduling: `weight`, `minShare`, `runningTasks`, `priority` and `stageId`. `runningTasks` is a runtime value and all other four are static properties of the Shedulable. However, TaskSetManager always has weight=1, minShare=0. Pool always has priority=0, stageId=-1(not sure why it’s defined as a var?).

The most important method in the Pool is **getSortedTaskSetQueue**. It returns all the TaskSetManager in the pool or its descendants in a sorted order. The implementation is sorting its children, and call each child’s getSortedTaskSetQueue recursively and concatenate the results together.

The tree structure of the pool and how the children are sorted depend on the `SchedulingMode`.

## FIFO
If the scheduling mode is FIFO, it’s just a single root pool and all the TaskSetManagers are in it. The tasks are sorted by `priority` first(from above, it’s probably the `jobId`), and then by stageId. Other fields are not used.

## FAIR
If the scheduling mode is FAIR, it’s more complicated. First there can be multiple pools. The pool structure is defined in a separate XML configuration file specified in `spark.scheduler.allocation.file`. See [offical doc](https://spark.apache.org/docs/latest/job-scheduling.html#scheduling-within-an-application) for details.

 Notice though the Schedulable allows deep nested structure, the allocation file only allow one level. That is: There’s a root pool, the root pool contains multiple pools defined in the file, and these pools will contain the TaskSetManagers. These pool can be either FIFO or FAIR themselves. The weight and minShare as well as the pool name can be specified for these pools. 

The pool the TaskSetManager is added to is decided by the property `SPARK_SCHEDULER_POOL`(`spark.scheduler.pool`) in TaskSetManager, which can be set using `SparkContext.setLocalProperty(SPARK_SCHEDULER_POOL, "pool_name")` before submitting the job.

### FAIR Sort Order
The sorted order is determined by runningTasks, minShare and weight.
* weight: This controls the pool’s share of the cluster relative to other pools. By default, all pools have a weight of 1. If you give a specific pool a weight of 2, for example, it will get 2x more resources as other active pools. Setting a high weight such as 1000 also makes it possible to implement priority between pools—in essence, the weight-1000 pool will always get to launch tasks first whenever it has jobs active. 
*  minShare: Apart from an overall weight, each pool can be given a minimum shares (as a number of CPU cores) that the administrator would like it to have. The fair scheduler always attempts to meet all active pools’ minimum shares before redistributing extra resources according to the weights. The minShare property can, therefore, be another way to ensure that a pool can always get up to a certain number of resources (e.g. 10 cores) quickly without giving it a high priority for the rest of the cluster. By default, each pool’s minShare is 0.

The sorting algorithm ensures every Schedule has at least minShare tasks running, secondly the running tasks are allocated according weight. The details are in **FairSchedulingAlgorithm**:
1. a Schedulable is "needy" if runningTasks < minShare. Needy ones will be served first.
2. If both are needy, lower minShareRatio=runningTasks/minShare goes first.
3. If neither is needy, lower taskToWeightRatio=runningTasks/weight goes first.
4. the name is used last to break even. (If still, the first one goes first)