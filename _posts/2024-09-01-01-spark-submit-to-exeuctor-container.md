---
title:   "Spark-Submit 이후 YARN 클러스터에서 실행되는 과정 알아보기"
excerpt: "Spark-Submit 이후 YARN 클러스터에서 실행되는 과정 알아보기"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - spark
  - hadoop
last_modified_at: 2024-09-01T:12:30+09:00
---

# Motivation

Apache Spark는 대규모 데이터를 효율적으로 처리하기 위한 분산 처리 프레임워크로, 데이터 엔지니어링과 분석 환경에서 널리 사용되고 있습니다. 다양한 클러스터 매니저와 호환되며, 그중에서도 YARN은 자주 활용되는 클러스터 매니저입니다. 그렇다면 Spark 애플리케이션을 YARN 클러스터에 제출하면, 내부에서는 어떤 일이 진행될까요?

이번 글에서는 `spark-submit` 명령어 이후 Spark와 YARN이 상호작용하며 `ApplicationMaster`와 `Driver`를 생성하고 `Executor` 컨테이너를 할당하는 과정을 자세히 살펴보고자 합니다. 이를 위해 Spark 애플리케이션이 실행되는 과정을 주요 클래스와 메서드 중심으로 분석해 보았습니다.

특히, 현업에서 자주 활용되는 YARN 클러스터 모드에 초점을 맞춰 Spark와 YARN이 어떻게 협력하는지, 그리고 그 과정에서의 핵심 흐름을 이해하기 쉽게 설명해 보려 합니다.
소스코드 분석에는 Spark version 3.5.3을 사용하였습니다.

# Contents

## Spark Submit

### [class] SparkSubmit

`SparkSubmit` 객체는 Spark 애플리케이션을 제출하고 실행하는 진입점입니다. 각 클러스터 매니저에 맞춰 `submit`할 클래스들을 정의후, 배포 모드(Client, Cluster)와 기타 설정에 맞춰 애플리케이션 실행을 관리합니다.

#### [method] runMain

`spark-submit` 명령어가 제출되면, 인자가 파싱된 후 `runMain` 메서드가 호출됩니다. 
`runMain`에서는 `prepareSubmitEnvironment` 메서드를 통해 Spark 설정, 메인 클래스와 같은 애플리케이션 환경이 설정됩니다. 
YARN의 클러스터 배포 모드로 제출되는 경우, `childMainClass`는 `org.apache.spark.deploy.yarn.YarnClusterApplication`으로 설정됩니다.
이후 해당 클래스의 인스턴스를 생성후 `app.start`를 통해 메인 메서드를 호출합니다.

```scala
def prepareSubmitEnvironment(args: SparkSubmitArguments, conf: Option[HadoopConfiguration] = None) : (Seq[String], Seq[String], SparkConf, String) = {
  if (isYarnCluster) {
    childMainClass = YARN_CLUSTER_SUBMIT_CLASS
  }
  (childArgs.toSeq, childClasspath.toSeq, sparkConf, childMainClass)
}

val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)
mainClass = Utils.classForName(childMainClass)

val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
  mainClass.getConstructor().newInstance().asInstanceOf[SparkApplication]
}

app.start(childArgs.toArray, sparkConf)
```

### [class] YarnClusterApplication

`YarnClusterApplication` 클래스에서는 YARN과 소통하는 `Client` 객체를 생성함으로써 에플리케이션 제출 및 실행을 관리합니다.
`Client` 클래스는 `ApplicationMaster`를 `ResourceManager`에 제출후 실행될 수 있도록 리소스 검증, 컨테이너 생성등의 기능을 수행합니다.

#### [method] submitApplication

YARN 클러스터와의 통신을 위해 `yarnClient`를 생성한후 상태를 모니터링할 수 있도록 `launcherBackend`와의 연결도 설정합니다.
이후 `yarnClient`을 통해 새로운 애플리케이션을 요청하여 AM 컨테이너 실행 준비를 합니다.

```scala
yarnClient.init(hadoopConf)
yarnClient.start()
launcherBackend.connect()
val newApp = yarnClient.createApplication()
```

`createContainerLaunchContext` 메소드를 통해 AM 컨테이너를 실행하기 위한 환경(클래스 경로, 실행 명령어 등)을 설정합니다. 
YARN에 클러스터 모드로 제출된 경우 `amClass`로 `org.apache.spark.deploy.yarn.ApplicationMaster`가 설정됩니다.
이후 `yarnClient.submitApplication`를 통해 애플리케이션을 YARN에 제출합니다.

```scala
def createContainerLaunchContext(): ContainerLaunchContext = {
  val amContainer = Records.newRecord(classOf[ContainerLaunchContext])
  val amClass =
    if (isClusterMode) {
      Utils.classForName("org.apache.spark.deploy.yarn.ApplicationMaster").getName
    } else {
      Utils.classForName("org.apache.spark.deploy.yarn.ExecutorLauncher").getName
    }
  amContainer.setCommands(amClass)
  amContainer
}

val containerContext = createContainerLaunchContext()
val appContext = createApplicationSubmissionContext(newApp, containerContext)

yarnClient.submitApplication(appContext)
```

## ApplicationMaster

### [class] ApplicationMaster

`Client` class에서 `ResourceManager`에 제출후 `AM container`에서 `ApplicationMaster`가 실행됩니다.
`ApplicationMaster` class는 유저 애플리케이션 실행, 모니터링을 하며 `executor` 자원할당을 관리합니다. 
`UserApplication`을 별도의 스레드에서 실행후 `SparkContext`가 성공적으로 생성되면 
이후 YARN에 AM을 등록후 `executor` container 자원을 요청 및 모니터링 합니다.

#### [method] run

`ClusterMode`인 경우 `runDriver` 메서드를 통해 YARN 클러스터 내에서 드라이버를 실행하여 애플리케이션을 제어합니다. 
`runDriver`와 다르게 `Client` 모드에서는 `runExecutorLauncher`가 실행되며 드라이버가 클러스터 외부에서 실행됩니다. 
`ClienMode`일 경우 `runExecutorLauncher`는 YARN에서 `executor`만 실행합니다.

```scala
if (isClusterMode) {
  runDriver()
} else {
  runExecutorLauncher()
}
```

#### [method] runDriver

`userApplication`을 별도의 스레드에서 실행합니다. 
여기서 애플리케이션의 유저 메인 클래스가 시작되며, 이 클래스는 클러스터에서 실행되는 드라이버 역할을 수행합니다.

```scala
userClassThread = startUserApplication()
```

`userApplication`에서 `SparkContext`가 초기화될 때까지 대기합니다. 
`SparkContext`가 정상적으로 설정되었는지 확인후, `ApplicationMaster`를 `registerAM`을 통해 YARN의 `ResourceManager`에 등록하여, 자원 할당 및 상태를 관리할 수 있게 합니다. 이후 리소스를 할당하는 `YarnAllocator`를 생성하여 `Executor` container 자원 할당 및 실행을 시작합니다.

```scala
val sc = ThreadUtils.awaitResult(sparkContextPromise.future,
  Duration(totalWaitTime, TimeUnit.MILLISECONDS))
if (sc != null) {
  val rpcEnv = sc.env.rpcEnv

  registerAM(host, port, userConf, sc.ui.map(_.webUrl), appAttemptId)

  createAllocator(driverRef, userConf, rpcEnv, appAttemptId, distCacheConf())
}
```

### [class] YarnAllocator

`YarnAllocator`는 `ResourceManager`에 `executor` container 자원을 요청하고 Locality(호스트, 랙, ANYHOST)순으로 선택한 다음 `executorRunnable`를 통해 `executor`를 실행시킵니다.

#### [method] allocateResource

`allocateResource`에서는 YARN `ResourceManager`에 자원 할당 요청을 보냅니다. 
`executor`를 실행할 컨테이너들을 `allocateResponse`에 받은후 `handleAllocatedContainers `메서드를 통해 할당된 컨테이너를 순회하며, 각 컨테이너에 `executor`를 배포합니다.

```scala
val allocateResponse = amClient.allocate(progressIndicator)
val allocatedContainers = allocateResponse.getAllocatedContainers()
handleAllocatedContainers(allocatedContainers.asScala.toSeq)
```

#### [method] handleAllocatedContainers

`containersToUse` 리스트로 실제 사용할 컨테이너를 저장하는 곳을 생성한후, 컨테이너와 호스트 매칭을 우선 시도합니다.
`matchContainerToRequest` 메서드를 통해 컨테이너의 호스트와 애플리케이션이 요청한 호스트와 일치하는지 확인합니다. 
일치하는 컨테이너는 위에서 선언된 `containersToUse`에 추가되고, 일치하지 않는 컨테이너는 이후 랙이 매칭되는지 확인합니다.

```scala
val containersToUse = new ArrayBuffer[Container](allocatedContainers.size)

val remainingAfterHostMatches = new ArrayBuffer[Container]
for (allocatedContainer <- allocatedContainers) {
  matchContainerToRequest(allocatedContainer, allocatedContainer.getNodeId.getHost,
    containersToUse, remainingAfterHostMatches)
}
```

컨테이너에 대해 랙 매칭을 다음 단계로 시도합니다.
`matchContainerToRequest`로 랙과 일치하는지 확인하고, 일치하는 컨테이너는 `containersToUse`에 추가됩니다.
일치하지 않는 경우 `ANY_HOST`를 사용하여 매칭을 시도하고, 이후 locality 기반으로 매칭되지 않은 불필요한 컨테이너들은 YARN `ResourceManager`에 반환됩니다.

호스트, 랙, 오프랙 순 매칭을 통해 `containerToUse`에 저장된 컨테이너들에 `runAllocatedContainers` 메서드를 통해
`executor`를 배치 및 실행합니다.


```scala
val remainingAfterRackMatches = new ArrayBuffer[Container]
if (remainingAfterHostMatches.nonEmpty) {
  for (allocatedContainer <- remainingAfterHostMatches) {
    val rack = resolver.resolve(allocatedContainer.getNodeId.getHost)
    matchContainerToRequest(allocatedContainer, rack, containersToUse,
      remainingAfterRackMatches)
  }
}

runAllocatedContainers(containersToUse)
```

#### [method] runAllocatedContainers

`executor` 실행을 위해 `ExecutorRunnable`을 사용합니다. `ExecutorRunnable`은 컨테이너 내에서 실행될 `executor`를 위한 자원(메모리, CPU)을 설정한후 `NodeManager`를 통해 `executor`를 실행하는 역할을 합니다. `launcherPool` 스레드풀을 이용해서 비동기적으로 여러 컨테이너에서 동시에 `executor`를 시작할 수 있게 해줍니다.

```scala
for (container <- containersToUse) {
  launcherPool.execute(() => {
    new ExecutorRunnable(
      Some(container),
      sparkConf,
      driverUrl,
    ).run()
    updateInternalState(rpId, executorId, container)
  })
}
```

### [class] ExecutorRunnable

`NodeManager` Client를 생성하여 `NodeManager`와 통신할 수 있도록 준비합니다. 이후 컨테이너의 실행 환경 객체를 생성후 `ClusterManager`에 맞게 실행 class를 생성후 `NodeManager`에 전달하여 `executor`를 실행합니다.

#### [method] run

`NMClient`를 생성하여 `NodeManager`와 통신할 수 있도록 준비합니다.
`startContainer` 메서드를 통해 컨테이너의 환경, 리소스를 설정후 실제로 `executor`를 실행합니다

```scala
nmClient = NMClient.createNMClient()
nmClient.init(conf)
nmClient.start()
startContainer()
```

#### [method] startContainer

`ContainerLaunchContext`는 컨테이너가 실행될 때 필요한 정보(자원, 환경 변수, 명령어 등)를 담고 있는 객체입니다. 
이 객체는 YARN `NodeManager`에게 전달되어 컨테이너의 실행 환경을 설정합니다.

```scala
val ctx = Records.newRecord(classOf[ContainerLaunchContext])
  .asInstanceOf[ContainerLaunchContext]
```

이후 `prepareCommand` 메서드를 통해 `executor`를 실행하기 위한 명령어(JAR 파일, 라이브러리)를 준비합니다. YARN에서 실행시 container 클래스로 `org.apache.spark.executor.YarnCoarseGrainedExecutorBackend`가 설정됩니다. `ctx.setCommands`를 통해 `ContainerLaunchContext`에 설정됩니다.

```scala
val commands = prepareCommand()
ctx.setLocalResources(localResources.asJava)
ctx.setCommands(commands.asJava)
```

설정된 `ContainerLaunchContext`를 YARN `NodeManager`에 전달하여, 할당된 컨테이너에서 실제로 `executor`를 실행하는 작업을 수행합니다.

```scala
nmClient.startContainer(container.get, ctx)
```

## Executor

### [class] CoarceGrainedExecutorBackend

`YarnCoarseGrainedExecutorBackend` 에서는 `CoarseGrainedExecutorBackend` 객체를 생성함으로써 `driver`와의 통신과 작업 처리를 담당합니다. `CoarseGrainedExecutorBackend` 클래스는 `executor`와 `driver` 간의 작업 전송과 상태 관리 기능을 수행하며, `executor`에서 발생하는 작업을 처리하고 그 결과를 `driver`에 전송하는 역할도 맡습니다.

#### [method] run

`setupEndpointRefByURI` 메서드를 사용하여 `driver`와의 RPC 연결을 위한 `EndpointRef`를 생성합니다.

```scala
val fetcher = RpcEnv.create("driverPropsFetcher", ... )
driver = fetcher.setupEndpointRefByURI(arguments.driverUrl)
```


`backendCreateFn`을 통해 `CoarceGrainedExecutorBackend` 객체를 `executor` 백엔드로 생성하고, 이를 RPC 엔드포인트로 등록합니다. 이로 인해 드라이버와의 통신이 활성화되며, 작업을 수신할 수 있게 됩니다.

```scala
val env = SparkEnv.createExecutorEnv(driverConf, ... )
val backend = backendCreateFn(env.rpcEnv, arguments, env, cfg.resourceProfile)
env.rpcEnv.setupEndpoint("Executor", backend)
```

#### [method] onStart

`CoarceGrainedExecutorBackend` 객체가 RPC 엔드포인트로 등록된후 `executor`가 준비되었음을 드라이버에게 알립니다. `asyncSetupEndpointRefByURI` 메서드로 RPC 엔드포인트를 통해 드라이버와의 연결을 설정합니다. 이후 `RegisterExecutor` 메시지를 드라이버에게 보내서, `executor`가 드라이버에 등록될 수 있도록 요청합니다. 이후 드라이버가 `executor`를 등록하면 `RegisteredExecutor` 메시지를 스스로에게 보내 `executor`가 정상적으로 등록되었음을 알립니다.

```scala
rpcEnv.asyncSetupEndpointRefByURI(driverUrl).flatMap { ref =>
  driver = Some(ref)
  ref.ask[Boolean](RegisterExecutor(executorId, self, hostname, cores, extractLogUrls,
    extractAttributes, _resources, resourceProfile.id))
}(ThreadUtils.sameThread).onComplete {
  case Success(_) =>
    self.send(RegisteredExecutor)
  case Failure(e) =>
    exitExecutor(1, s"Cannot register with driver: $driverUrl", e, notifyDriver = false)
}(ThreadUtils.sameThread)
```

#### [method] receive

`executor`에서 RPC 요청을 받았을때 호출되는 메서드 입니다. `RegisteredExecutor` 메시지는 드라이버에 `executor`가 성공적으로 등록되었을 때 수신됩니다. `executor`가 등록되면 `task`를 수행하는 `Executor` 객체를 생성하고, 작업을 처리할 준비가 되었음을 알리는 `LaunchedExecutor` 메시지를 드라이버에게 전송합니다.

```scala
case RegisteredExecutor =>
  logInfo("Successfully registered with driver")
  try {
    executor = new Executor(executorId, hostname, env, getUserClassPath, isLocal = false,
      resources = _resources)
    driver.get.send(LaunchedExecutor(executorId))
  } catch {
    case NonFatal(e) =>
      exitExecutor(1, "Unable to create executor due to " + e.getMessage, e)
  }
```

# Conclusion

이번 분석에서는 `spark-submit` 명령어 실행부터 시작해, `ApplicationMaster` 컨테이너가 실행되고, YARN `ResourceManager`와 `NodeManager`를 통한 요청으로 `Executor` 컨테이너가 할당되고 `ExecutorBackend`가 시작되는 전체 과정을 살펴보았습니다.

특히, `YarnAllocator`가 컨테이너를 할당할 때 호스트, 랙, ANYHOST 등의 locality 우선순위를 기반으로 `Executor `서버를 선택하는 과정이 인상적이었습니다. 이 과정은 이후 `Driver`의 `TaskScheduler`가 `Task`를 `Executor`에 분배할 때 locality를 고려해 데이터 셔플을 최소화하기 위한 사전 작업임을 확인할 수 있었습니다.

# Reference

---

[https://github.com/apache/spark/tree/master](https://github.com/apache/spark/tree/master)