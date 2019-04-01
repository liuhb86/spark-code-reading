# How are the results sent back to the application?
For inter-stage shuffle tasks, `MapStaus` are returned from the `runTask()` function. (The detail will be covered later) For the final `ResultTask`, the result of the supplied function will be returned to the Executor’s `TaskRunner.run()`. It will be serialized by an instance of `SparkEnv.serializer`. A `DirectTaskResult` object is created with the serialized bytes along with the accumulators (covered later), and is serialized again with `closureSerializer`. How it’s sent back depends on  the size of the serialized data.

* If it’s larger than `maxResultSize`(`spark.driver.maxResultSize`, default is 1G), the result will be dropped;
* If it’s larger than `maxDirectResultSize`(minimium of `spark.task.maxDirectResultSize`, default is 1M, and `spark.rpc.message.maxSize`, default is 128M), the result is put to `BlockManager` and the `blockId` ("taskresult_" + `taskId`) is returned as the result.
* Otherwise, the result can be sent back directly.
* 
The `ExecutorBackend` is notified by `execBackend.statusUpdate(taskId, TaskState.FINISHED, serializedResult)`, the parameters will finally reach `TaskScheduler.statusUpdate`. For local mode, it’s returned to `LocalSchedulerBackend`(which is also a `ExecutorBackend`) -–(local RPC)--> `LocalEndpoint` --> `TaskScheduler.statusUpdate`. For standalone mode, the path is `CoarseGrainedExecutorBackend` -–(RPC from executor to driver)--> `StandaloneSchedulerBackend.DriverEndpoint`--> `TaskScheduler.statusUpdate`.

In `TaskScheduler.statusUpdate`, first, the task is remove from some of the internal maps, suc as `taskIdToTaskSetManager`, `taskIdToExecutorId`, `executorIdToRunnngTaskIds`. Then, the task is removed from its `TaskSetManager` and the number of running task of the parent pools are updated recursively. Then `TaskResultGetter.enqueueSucessfulTask` is called.

The `TaskResultGetter` has a threadpool:` ThreadUtils.newDaemonFixedThreadPool` and 4 threads by default(`spark.resultGetter.threads`). The result will be deserialized by one of the worker threads. If it’s large indirect result, the result will be got from the `BlockManager.getRemoteBytes` and then deserialized. If there are updates to accumulators, they will also be updated(details later). The it calls back `TaskScheduler.handleSuccessfulTask` which just calls `TaskSetManager.handleSuccessfulTask`.

`TaskSetManager.handleSuccessfulTask` will mark the task as successful if everything goes well, and if  all tasks in the `TaskSetManager` are successful, the `isZombie` of the `TaskSetManager` will be set to true. 

Next, `TaskScheduler.markPartitionCompleted` is called. Accroding to the comments,

 > "There may be multiple tasksets for this stage -- we let all of them know that the partition was completed.  This may result in some of the tasksets getting completed.”

 But I don’t know when it will happen.??? (See [SPARK-23433](https://issues.apache.org/jira/browse/SPARK-23433), see also [25250](https://issues.apache.org/jira/browse/SPARK-25250),[26634](https://issues.apache.org/jira/browse/SPARK-26634)). 
 
 Then, `DAGScheduler.taskEnded` is called and then the tasksetManager is removed from pool if all tasks are finished.

In `DAGScheduler.taskEnded/handleTaskCompletion`, we will ignore the part for updating accumulators and handling shuffle tasks. We will cover them later. For shuffle tasks, it will check if all the tasks in the shuffle stage are finished, and if so, it will resubmit the pending child stages to continue the downstream execution.

Here, we only looks the process for `ResultTask`. It will update the finished partitions of the associate `ActiveJob` and if the all the partitions are finished, it will mark the stage complete and do cleanups. Ultimately, it will call `Job.jobListener`(which is usually a `JobWaiter`)`.taskSucceed()`. It will call back the `resultHandler` passed in, and if all tasks are finished, it will fulfill the `jobPromise`. That is the end of the job execution. 

Recall the [Converting RDD to Job](./rdd2job.md) section. The series of overloaded `runJob` functions are used to construct proper `resultHandler` to consume the result. They will create an array of length of partitions. They take a function parameter `func` to be applied on the result of each partition.The `resultHandler` they construct will apply this function to the task result, and then save the value into the corresponding slot of the array. After the job is finished, the array with result of each partition will be returned to the RDD action, which just combine the array to the final result.

E.g., for `count()`, the `func` is simply 
```scala
Utils.getIteratorSize(iterator: Iterator[_]): Long = {
    var count = 0L
    while (iterator.hasNext) {
        count += 1L
        iterator.next()
    }
        count
}
```

And the array will be summed together.

For `collect()`, the `func` is simply `iter.toArray`. And the array of array will be concatenated together.
