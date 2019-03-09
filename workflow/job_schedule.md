# How are jobs and stages scheduled?

Submit status of stage:
* Waiting
* Running
* Failed -- The three above are maintained in sets waitingStages, runningStages and failedStages of DAGScheduler
* (not submitted)

After the stage tree is built, an **ActiveJob** object is created. Then `submitStage()` is called.

First, it checks whether all parent (dependent) shuffle stages are completed (*available*). 
A shuffle stage is available if the `availableOuputs` from `mapOutputTrackerMaster`(This will be covered in the shuffle section) equals to the number of partitions. (`ShuffleMapStage.isAvailable`)
If missing stages are found, the missing stages are submitted recursively. And the current stage is put into the waiting set.

If there are no missing stages, submitMissingTask is called to running the tasks inside the stage, and the stage will be put into the running set.

TODO: follow up