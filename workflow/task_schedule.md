# How are tasks scheduled?
It starts from `DAGScheduler.submitMissingTasks`.

First, find all missing partitions of this stage. For final stage, the information is maintained in the ActiveJob; For shuffle stages, it’s maintained in the `mapOutputTrackerMaster`.

Then, `makeNewStageAttempt` will create a new StageInfo in the Stage.

Then, the RDD along with the dependencies are serialized with `clousureSerializer` and broadcast(It seems it will broadcast to all nodes, not only where the task will be executed?). (the serialization and broadcast will be covered in separate sections.)

Then, the **Tasks** are created for each missing partition, either a **ShuffleMapTask** or **ResultTask**.  The preferred location from the RDD is passed to the tasks.

Then the tasks are submitted to `TaskScheduler.submitTasks` bundled in a **TaskSet**. Notice that the jobId is passed in as the priority. So it’s kind of FIFO.

The following part needs some background knowledges:
1. [Spark RPC](../infrastructure/rpc.md) The first section of the reference article is a must read.
2. The [scheduler pool](../infrastructure/pool.md).

After understanding the Pool, we can go back to the TaskScheduler.submitTasks.

***TODO*** formatting.

The **TaskScheduler** is another important class for scheduling. It schedules the tasks as the name indicated. There’s only one implementation of the trais TaskScheduler, which is TaskSchedulerImpl. Below, we will refer it as TaskScheduler for short. Also see https://jaceklaskowski.gitbooks.io/mastering-apache-spark/spark-TaskSchedulerImpl.html.



In submitTasks, first a TaskSetManager is created for managing the TaskSet. The TaskScheduler maintains a map taskSetsByStageIdAndAttempt: stageId -> AttemptId -> TaskSetManager.

Then the TaskSetManager is add to the pool by the SchedulableBuilder. addTaskSetManager. 
 

After the tasks is added, backend.reviveOffers() is called for the next step.

The backend is a subclass of SchedulerBackend trait. Because Spark support different cluster environment, schedulerBackend is designed to abstract from the implementation details for different clusters and provided a unified interface to interact with TaskSchduler. Each cluster environment has its own SchedulerBackend implementation, such as LocalSchedulerBackend, StandaloneSchedulerBackend, MesosCoarseGrainedSchedulerBackend, etc.
The SchedulerBackend has a few abstract methods, such as start, stop, defaultParallelism, maxNumConcurrencyTask, etc. 
But the most important one is reviveOffers. It will send the information of available resources in each of the executors it manages to the TaskScheduler via TaskScheduler.resourceOffers, and TaskScheduler will return its decision to the SchedulerBackend and the SchedulerBackend will assign the tasks to the executors. 
Notice that the calls are not straightforward. Each SchedulerBackend registers a corresponding RpcEndPoint in the RpcEnv and in the reviveOffers(), it makes a “local” RPC call and the real logic of reviveOffers is in the RpcEndpoint. The documentation says using RPC call is to avoid deadlock(LocalSchedulerBackend.scala):
Calls to [[LocalSchedulerBackend]] are all serialized through LocalEndpoint. Using an RpcEndpoint makes the calls on [[LocalSchedulerBackend]] asynchronous, which is necessary to prevent deadlock between [[LocalSchedulerBackend]] and the [[TaskSchedulerImpl]]. 
But I’m not sure what the exact reason is.
The flow is like this:

***TODO***