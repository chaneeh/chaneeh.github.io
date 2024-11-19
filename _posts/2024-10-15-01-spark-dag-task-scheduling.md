---
title:   "Spark 내부 동작 분석: RDD Action 실행에서 Task 분배까지"
excerpt: "Spark 내부 동작 분석: RDD Action 실행에서 Task 분배까지"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - spark
  - hadoop
last_modified_at: 2024-10-15T:12:30+09:00
---

# Motivation

Spark는 대규모 데이터를 분산 처리하기 위한 프레임워크로, 빠른 데이터 처리와 효율적인 작업 스케줄링을 제공하고 있습니다. Spark 애플리케이션을 작성하며 RDD에 액션을 적용할 때, 내부적으로 어떤 일이 일어나는지 궁금했던 적이 있으신가요?

이번 글에서는 RDD 액션 호출 이후 Spark 내부에서 DAG(Directed Acyclic Graph)가 생성되고, 이를 기반으로 Task가 만들어져 Executor에 분배되기까지의 과정을 살펴보려 합니다. 이 과정에서 Spark의 주요 컴포넌트(`SparkContext`, `DAGScheduler`, `TaskSchedulerImpl`, `CoarseGrainedSchedulerBackend`)가 각각 어떤 역할을 수행하는지도 확인해보고자 합니다.

특히, RDD의 Dependency를 기반으로 DAG를 생성하고 Task를 Scheduling하는 Spark의 내부 과정에 초점을 맞추었습니다. 이 글을 통해 Spark의 내부 구조를 파악하여 더 깊이 있는 활용에 도움이 되면 좋겠습니다. 소스코드 분석에는 spark version 3.5.3을 사용하였습니다.

# Contents

## 1. RDD Action 호출: Stage와 Task가 만들어지는 과정

### [class] SparkContext

Spark 애플리케이션의 시작점으로, 사용자 코드에서 호출되는 `Action` 메서드의 요청을 처리합니다. `SparkContext`는 애플리케이션 실행에 필요한 class 환경을 설정하고, RDD에 대한 Action 요청을 `DAGScheduler`로 전달합니다.

#### [method] submitJob

사용자 코드에서 `Action`이 호출될 때 `SparkContext`에서 처음 호출되는 method 입니다.
`RDD`, 파티션 정보와 같은 세부정보를 담아 `DAGScheduler.submitJob`에게 전달합니다.

```scala
def submitJob[T, U, R](
    rdd: RDD[T],
    processPartition: Iterator[T] => U,
    partitions: Seq[Int],
    resultFunc: => R): SimpleFutureAction[R] =
{
    assertNotStopped()
    val cleanF = clean(processPartition)
    val callSite = getCallSite()
    val waiter = dagScheduler.submitJob(
        rdd,
        (context: TaskContext, iter: Iterator[T]) => cleanF(iter),
        partitions,
        callSite,
        localProperties.get)
    new SimpleFutureAction(waiter, resultFunc)
}
```

### [class] DAGScheduler

`RDD`의 의존성을 기반으로 `DAG`(Directed Acyclic Graph)를 생성하고, 이를 Stage로 분리한 후, 각 Stage를 `Task` 단위로 나눕니다. `TaskSchedulerImpl`에 `Task`를 전달해 실제 `Executor`에서 작업이 실행되도록 처리합니다.

#### [method] submitJob

`SparkContext`에서 받은 요청을 처리하여 이벤트 큐(`eventProcessLoop`)에 `JobSubmitted` 이벤트를 추가하여 `DAG` 생성 및 `Task` 분배를 준비합니다.

```scala
def submitJob[T, U](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int],
    callSite: CallSite,
    properties: Properties): JobWaiter[U] =
{
    val jobId = nextJobId.getAndIncrement()
    val waiter = new JobWaiter[U](this, jobId, partitions.size)
    eventProcessLoop.post(JobSubmitted(
      jobId, rdd, func2, partitions.toArray, callSite, waiter,
      JobArtifactSet.getActiveOrDefault(sc),
      Utils.cloneProperties(properties)))
    waiter
}
```


#### [method] createResultStage

최종적으로 실행될 `ResultStage`를 생성하여 stage submit을 준비하는 method입니다.
`getOrCreateParentStages` 메서드를 통해 parent 관계에 있는 `ShuffleMapStage`들을 정의합니다.
parent stage들과 `ResultStage`를 통해 `DAG`를 구성합니다.

```scala
private def createResultStage(
    rdd: RDD[_],
    func: (TaskContext, Iterator[_]) => _,
    partitions: Array[Int],
    jobId: Int,
    callSite: CallSite): ResultStage = 
{
    val (shuffleDeps, _) = getShuffleDependenciesAndResourceProfiles(rdd)
    val parents = getOrCreateParentStages(shuffleDeps, jobId)
    val id = nextStageId.getAndIncrement()
    val stage = new ResultStage(id, rdd, func, partitions, parents, jobId, callSite)
    stage
}
```


#### [method] getOrCreateParentStages

위의 `ResultStage`에서 `DAG`구성에 필요한 parent stage들을 생성하기 위해 호출되는 method 입니다.
`Action`이 호출된 `RDD`의 `ShuffleDependency`를 기반으로  Parent `ShuffleMapStage`들을 생성합니다.

```scala
private def getOrCreateParentStages(
    shuffleDeps: HashSet[ShuffleDependency[_, _, _]],
    firstJobId: Int): List[Stage] = 
{
    shuffleDeps.map { shuffleDep =>
    getOrCreateShuffleMapStage(shuffleDep, firstJobId)
    }.toList
}
```


#### [method] submitStage

위 단계에서 생성된 `ResultStage`를 시작으로, parent Stage들의 실행을 재귀적으로 처리합니다. 
parent Stage가 완료될 때까지 대기하여 `DAG`의 의존성을 만족한 후에 각 stage는 `submitMissingTasks`에 전달되어 `Task`로 분리됩니다.

```scala
private def submitStage(stage: Stage): Unit = {
    val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
      if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
        val missing = getMissingParentStages(stage).sortBy(_.id)
        if (missing.isEmpty) {
          submitMissingTasks(stage, jobId.get)
        } else {
          for (parent <- missing) {
            submitStage(parent)
          }
          waitingStages += stage
        }
    }
    }
}
```

#### [method] submitMissingTasks

실행가능한 `Stage`를 `Task` 단위로 나누는 작업을 처리합니다. `Stage` 유형에 따라 `Task`를 `ShuffleMapTask` 또는 `ResultTask`로 생성한후, `Task`를 `Executor`에 스케쥴링 하는 `TaskSchedulerImpl`에 전달합니다.

```scala
private def submitMissingTasks(stage: Stage, jobId: Int): Unit = {
    val tasks: Seq[Task[_]] = try {
      stage match {
        case stage: ShuffleMapStage =>
          partitionsToCompute.map { id =>
          new ShuffleMapTask(stage.id, stage.latestInfo,
            stage.numPartitions, taskIdToLocations(id), stage.rdd.isBarrier())
          }

        case stage: ResultStage =>
          partitionsToCompute.map { id =>
          val part = partitions(id)
          new ResultTask(stage.id, stage.latestInfo,
            stage.numPartitions, taskIdToLocations(id), id, stage.rdd.isBarrier())
          }
      }
    }
    taskScheduler.submitTasks(new TaskSet(tasks.toArray, stage.id, ...))
}
```

## 2. Task Scheduling: Executor로 작업이 전달되는 과정

### [class] TaskSchedulerImpl

`DAGScheduler`에서 전달받은`TaskSet`을 스케줄링하여 `Executor`에 작업을 할당합니다. Locality 수준(`Process Local`, `Node Local`, `Rack Local`, `ANY`)에 따라 가장 적합한 `Executor`를 선택합니다.

#### [method] resourceOffers

`Executor`로부터 제공받은 리소스를 기반으로 `Task`들을 스케쥴링하는 method 입니다. 
`Taskset`을 순회하며 데이터 locality 에 가장 적합하게 `Executor`를 배정합니다. 
locality의 우선순위는 `Process Local` → `Node Local` → `Rack Local` → `ANY`순 입니다.

```scala
def resourceOffers(
    offers: IndexedSeq[WorkerOffer]): Seq[Seq[TaskDescription]] = synchronized {
  for (taskSet <- sortedTaskSets) {
    for (currentMaxLocality <- taskSet.myLocalityLevels) {
        var launchedTaskAtCurrentMaxLocality = false
        do {
        val (noDelayScheduleReject, minLocality) = resourceOfferSingleTaskSet(
            taskSet, currentMaxLocality, availableCpus, availableResources)
        launchedTaskAtCurrentMaxLocality = minLocality.isDefined
        launchedAnyTask |= launchedTaskAtCurrentMaxLocality
        } while (launchedTaskAtCurrentMaxLocality)
    }
  }
  tasks.map(_.toSeq)
}
```

### [class] CoarseGrainedSchedulerBackend

`Executor`와 `Driver` 간의 통신을 관리하며, `Task`를 `Executor`로 전송하는 역할을 합니다.

#### [method] launchTasks

위 단계에서 생성된 `TaskDescription`을 기반으로 각 `Executor`에게 `Task`를 전달합니다. 
`TaskDescription`정보를 직렬화 하여 RPC 호출(`LaunchTask`)을 수행합니다. 
이후 `ExecutorBackend`에서 RPC 호출(`LaunchTask`)을 수신하여 `Task`를 실행합니다.

```scala
private def launchTasks(tasks: Seq[Seq[TaskDescription]]): Unit = {
  for (task <- tasks.flatten) {
    val serializedTask = TaskDescription.encode(task)
    val executorData = executorDataMap(task.executorId)
          
    executorData.freeCores -= task.cpus
    task.resources.foreach { case (rName, addressAmounts) =>
    executorData.resourcesInfo(rName).acquire(addressAmounts)
    }
    executorData.executorEndpoint.send(LaunchTask(new SerializableBuffer(serializedTask)))
  }
}
```


# Conclusion

이 글에서는 Spark 애플리케이션이 실행될 때 RDD의 Action 호출로부터 시작해 DAG를 생성하고, Task로 분해해 Executor에 전달되는 전 과정을 살펴보았습니다. Spark의 주요 컴포넌트들이 어떻게 협력하는지와 함께 RDD → DAG 생성 → Task Scheduling으로 이어지는 내부 동작을 소스코드를 통해 확인하며, Spark의 효율적인 설계 원리를 이해할 수 있었습니다.

특히, Task Scheduling 과정에서 데이터 Locality를 기반으로 최적화하는 부분은, 클러스터 자원을 효과적으로 활용하기 위한 spark의 설계를 확인할수 있어서 신기하였습니다.

# Reference

---

[https://github.com/apache/spark](https://github.com/apache/spark)