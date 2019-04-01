How does driver programs communicate with the master (cluster manager)?

First, notice the following section from Spark doc:
https://spark.apache.org/docs/latest/cluster-overview.html
There are several useful things to note about this architecture:
    1. Each application gets its own executor processes, which stay up for the duration of the whole application and run tasks in multiple threads. This has the benefit of isolating applications from each other, on both the scheduling side (each driver schedules its own tasks) and executor side (tasks from different applications run in different JVMs). However, it also means that data cannot be shared across different Spark applications (instances of SparkContext) without writing it to an external storage system.
    2. Spark is agnostic to the underlying cluster manager. As long as it can acquire executor processes, and these communicate with each other, it is relatively easy to run it even on a cluster manager that also supports other applications (e.g. Mesos/YARN).
    3. The driver program must listen for and accept incoming connections from its executors throughout its lifetime (e.g., see spark.driver.port in the network config section). As such, the driver program must be network addressable from the worker nodes.
    4. Because the driver schedules tasks on the cluster, it should be run close to the worker nodes, preferably on the same local area network. If you’d like to send requests to the cluster remotely, it’s better to open an RPC to the driver and have it submit operations from nearby than to run a driver far away from the worker nodes.

During SparkContext creation, the TaskScheduler and its SchedulerBackend are created and started. 
For standalone cluster, the SchedulerBackend is StandaloneSchedulerBackend. It is extends from CoarseGrainedSchedulerBackend, which is also used by many other scheduler backends. So both classes are important. 

In StandaloneSchedulerBackend.start(), it construct an **ApplicationDescription**, which contain properties of this application:
* name: it's a parameter when creating the SparkContext
* maxCores: the maximum amount of CPU cores to request for the application from across the cluster. It's from conf [`spark.cores.max`](https://spark.apache.org/docs/latest/configuration.html#scheduling).  If not set, the default will be `spark.deploy.defaultCores` on Spark's standalone cluster manager, whose default is infinite. (See https://spark.apache.org/docs/latest/spark-standalone.html)
* memoryPerExecutorMB: from conf `spark.executor.memory`. If not set, from System variable `SPARK_EXECUTOR_MEMORY`. If not set, default is 1GB.
* command: This is the java command line for launching executors. It also contains several fields:
    * mainClass: the Executor class. `org.apache.spark.executor.CoarseGrainedExecutorBackend`
    * arguments: 
        * "--driver-url": "spark://$name@${rpcAddress.host}:${rpcAddress.port}"
            * host: `spark.driver.host`. If not set, got from `Utils.localCanonicalHostName()`
            * port: `spark.driver.port`, if not set, default is 0?
            * name: RPC endpoint name, "CoarseGrainedScheduler"
        * "--executor-id": "{{EXECUTOR_ID}}",
        * "--hostname": "{{HOSTNAME}}",
        * "--cores": "{{CORES}}",
        * "--app-id": "{{APP_ID}}",
        * "--worker-url": "{{WORKER_URL}}")
    * environment: Environment variables to pass to our executors. See `executorEnvs` related code in `SparkContext`.
    * classPathEntries: `spark.executor.extraClassPath`. This exists primarily for backwards-compatibility with older versions of Spark. Users typically should not need to set this option. 
    * libraryPathEntries: `spark.executor.extraLibraryPath`
    * javaOpts: `Utils.sparkJavaOpts` + `spark.executor.extraJavaOptions`
* appUiUrl: for SparkUI, we will ignore it.
* eventLogDir: not essential, we will ignore it.
* eventLogCodec: not essential, we will ignore it.
* coresPerExecutor: The number of cores to use on each executor. `spark.executor.cores`. See [Executors Scheduling](https://spark.apache.org/docs/latest/spark-standalone.html#executors-scheduling) and [Task Scheduling](../workflow/task_schedule.md)
* initialExecutorLimit: number of executors this application wants to start with, only used if dynamic allocation is enabled
* user: from System Property `user.name`. Default is "\<unknown\>".