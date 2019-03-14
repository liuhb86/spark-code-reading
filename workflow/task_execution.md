# How does executor execute tasks?
The **Executor** class has a threadpool created by Java’s ExecutorService (here’s one [article]( https://www.baeldung.com/java-executor-service-tutorial) about it) `Executors.newCachedThreadPool`. 

When `Executor.launchTask` is called, a new     `TaskRunner`(which is a Runnable) is created and it’s put to the threadpool to execute. The `TaskRunner` is also put in the `runningTasks` for management.

The TaskRunner takes two parameters: `executorBackend` for sending back status update, and the `taskDescription`. 

It first deserialize the task. It setup the ClassLoader, and adds the dependent jars and files to the class loader. Then a `clousureSerializer` is created and the task is deserialized.

`Task.run()` is called afterwards. It calls `Task.runTask`, which is override by each of its sub classes. 

For `ResultTask`, it basically deserialize the RDD and the final function from the `taskBinary`, and call the function with `RDD.iterator`. If RDD is cached or checkpointed, it will iterate from the cached data(covered later). Otherwise, it will call the **compute()** function, which is overrided by each RDD,  to do the real work. 
For shuffled RDD, the shuffled data will be read from `SparkEnv.shuffleManager`(covered later), so it grantee the task won’t cross the stage.

For `ShuffledTask`, the RDD is iterated similarly, but the result will be write to `ShuffleManager`(covered later).

