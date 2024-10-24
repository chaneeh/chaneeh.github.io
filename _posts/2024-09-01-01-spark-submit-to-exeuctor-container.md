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


Spark 애플리케이션을 제출하고 나면 실제로 YARN 클러스터 내부에서 무슨 일이 벌어질까요?
 
이번 글은 spark-submit 이후 Spark와 YARN이 어떻게 상호작용하며 Executor 컨테이너를 할당하는지, 그리고 driver와 executor가 어떻게 소통하는지에 대해 더 자세하게 알고싶어서 작성하게 되었습니다.

주요 실행 환경, 클래스, 그리고 method 중심으로 소스코드를 살펴보았고 현업에서 자주 쓰이는 yarn의 cluster 모드 위주로 분석하였습니다.

# Contents

## Spark Submit

### [class] SparkSubmit

SparkSubmit 객체는 Spark 애플리케이션을 제출하고 실행하는 진입점입니다. 각 클러스터 매니저에 맞춰 submit할 클래스들을 정의후, 배포 모드(Client, Cluster)와 기타 설정에 맞춰 애플리케이션 실행을 관리합니다.

```scala
object SparkSubmit extends CommandLineUtils with Logging {
  def main(args: Array[String]): Unit = { ... }
}

class SparkSubmit extends Logging {
  def doSubmit(args: Array[String]): Unit = { ...  }
  def submit(args, ...): Unit = { ... }
  def prepareSubmitEnvironment() : Seq[String] = { ... }
  def runMain(args, uninitLog): Unit = { ... }
}
```

#### [method] main

Spark 애플리케이션 실행의 진입점인 method에서는 SparkSubmit 객체를 생성하고, 명령줄 인자를 파싱한 후 애플리케이션을 제출하는 doSubmit() 메서드를 호출합니다. 

```scala
val submit = new SparkSubmit() { ... }
submit.doSubmit(args)
```

#### [method] doSubmit

사용자가 명령줄 인자를 파싱후 요청한 작업(애플리케이션 제출, 종료, 상태 요청)을 처리합니다.
사용자가 spark-submit 명령어를 통해 애플리케이션을 제출하고자 할 경우, submit() 메서드를 호출하여 클러스터에 애플리케이션을 제출합니다.

```scala
val appArgs = parseArguments(args)

appArgs.action match {
  case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog, sparkConf)
  case SparkSubmitAction.KILL => kill(appArgs, sparkConf)
  case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs, sparkConf)
  case SparkSubmitAction.PRINT_VERSION => printVersion()
}
```

#### [method] submit

submit() 메서드는 사용자가 제출한 애플리케이션의 실행 모드(Client/Cluster)에 맞춰 적절한 환경에서 애플리케이션을 실행합니다. Kubernetes와 같은 리소스 매니저와 연동될 때의 프록시 사용자 처리를 하거나 standalone 클러스터일 경우 RPC 또는 REST 프로토콜을 사용해 애플리케이션을 제출합니다.
이후 runMain() 메서드를 통해 파싱된 명령줄 인자를 기반 애플리케이션의 메인 클래스를 실행합니다.

```scala
if (isKubernetesClusterModeDriver) { 
  // Kubernetes 클러스터에서 Client 모드일때 proxy user 권한으로 실행
}
if (args.isStandaloneCluster && args.useRest) {
  // Standalone 클러스터에서 제출시 REST, RPC 방식 시도
}
runMain(args, uninitLog)
```

#### [method] runMain

prepareSubmitEnvironment() 메서드 통해 애플리케이션의 환경을 설정합니다. 여기에는 애플리케이션 인자, 클래스 경로, Spark 설정, 메인 클래스 등이 포함됩니다. yarn에 cluster mode로 제츨시 childMainClass는 `org.apache.spark.deploy.yarn.YarnClusterApplication` 로 설정 됩니다. 

```scala
val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)
```

애플리케이션의 메인 클래스를 로드한후 로드된 메인 클래스가 SparkApplication 인터페이스를 구현하는지 확인한 후, 해당 클래스의 인스턴스를 생성합니다. 만약 Java 메인 클래스일 경우, 이를 JavaMainApplication으로 감싸서 실행합니다.

```scala
mainClass = Utils.classForName(childMainClass)

val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {
  mainClass.getConstructor().newInstance().asInstanceOf[SparkApplication]
} else {
  new JavaMainApplication(mainClass)
}
```

최종적으로 애플리케이션의 메인 메서드를 호출하여 실행을 시작합니다. 이때, 애플리케이션 인자(childArgs)와 Spark 설정(sparkConf)이 전달됩니다.

```scala
app.start(childArgs.toArray, sparkConf)
```


### [class] YarnClusterApplication

YarnClusterApplication 클래스는 YARN 클러스터 모드에서 애플리케이션을 실행하기 위한 진입점 클래스입니다. Client 객체를 생성함으로써 에플리케이션 제출 및 실행을 관리합니다.

```scala
class YarnClusterApplication extends SparkApplication {
  override def start(args: Array[String], conf: SparkConf): Unit = {
    new Client(new ClientArguments(args), conf, null).run()
  }
}
```


### [class] Client

Client 클래스는 YARN 클러스터에서 Spark 애플리케이션을 제출하고 실행하는 데 필요한 과정을 처리하는 클래스입니다. 이 클래스는 애플리케이션 AM을 ResourceManager에 제출후 실행될 수 있도록 리소스 검증, 컨테이너 생성, 모니터링 등의 기능을 수행합니다.

```scala
class Client(args: ClientArguments, sparkConf: SparkConf, rpcEnv: RpcEnv) extends Logging {
  def run(): Unit = { ... }
  def submitApplication(): Unit = { ... }
}
```

#### [method] run

submitApplication() 메서드를 호출하여 애플리케이션을 YARN 클러스터에 제출합니다. 이 단계에서 YARN의 ResourceManager에게 애플리케이션을 제출하고, YARN이 애플리케이션 컨테이너를 생성하여 실행을 시작합니다.

```scala
submitApplication()
```

monitorApplication() 메서드를 호출하여 애플리케이션이 YARN에서 실행되는 동안 상태를 지속적으로 모니터링합니다. 이 메서드는 애플리케이션의 상태가 종료될 때까지 주기적으로 상태를 확인합니다. 이후 성공적으로 처리되지 않았다면 SparkException을 발생시킵니다.

```scala
val YarnAppReport(appState, finalState, diags) = monitorApplication()
```

#### [method] submitApplication

YARN 클러스터와의 통신을 위해 yarn 클라이언트를 초기화합니다. 
애플리케이션 제출 후 상태를 모니터링할 수 있도록 런처 백엔드와의 연결도 설정합니다. 

```scala
launcherBackend.connect()
yarnClient.init(hadoopConf)
yarnClient.start()
```

YARN ResourceManager에 새로운 애플리케이션을 요청하고 모티터링을 위해 애플리케이션 ID를 가져옵니다. 이 애플리케이션 ID는 클러스터 내에서 애플리케이션을 식별하는 고유한 값입니다.
아직 애플리케이션이 제출된 상태는 아니며, ApplicationID 와 SubmissionContext를 생성하였습니다. 이후 applicationSubmissionContext를 설정후 submit을 하게 됩니다.

```scala
val newApp = yarnClient.createApplication()
val newAppResponse = newApp.getNewApplicationResponse()
this.appId = newAppResponse.getApplicationId()
```

createContainerLaunchContext 메소드를 통해 AM 컨테이너를 실행하기 위한 환경(클래스 경로, 실행 명령어 등)을 설정합니다. yarn에 클러스터 모드로 제출된 경우 AM class로 `org.apache.spark.deploy.yarn.ApplicationMaster` 가 설정됩니다.
이후 createApplicationSubmissionContext를 통해 YARN에 애플리케이션을 제출할 때 사용할 애플리케이션 제출 컨텍스트를 생성합니다.

```scala
val containerContext = createContainerLaunchContext()
val appContext = createApplicationSubmissionContext(newApp, containerContext)
```

최종적으로 애플리케이션을 YARN에 제출하고, 애플리케이션이 성공적으로 제출되었음을 런처 백엔드에 보고합니다. 제출이 완료되면 launcherBackend를 통해 appID를 SUBMITTED 상태로 변경합니다.

```scala
yarnClient.submitApplication(appContext)
launcherBackend.setAppId(appId.toString)
reportLauncherState(SparkAppHandle.State.SUBMITTED)
```

## ApplicationMaster

### [class] ApplicationMaster

client 클래스에서 RM에 제출후 AM container에서 ApplicationMaster가 실행됩니다.
ApplicationMaster class는 유저 애플리케이션 상태 실행, 모니터링을 하며 executor 자원할당을 관리합니다. userApplication을 별도의 스레드에서 실행후 sparkcontext가 성공적으로 생성되면 이후 yarn에 am을 등록후 executor container 자원을 요청 및 모니터링 합니다.

```scala
class ApplicationMaster(args: ApplicationMasterArguments, sparkConf: SparkConf, yarnConf: YarnConfiguration) extends Logging {
  def run(): Int = { ... }
  def runDriver(): Unit = { ... }
}
```

#### [method] run

cluster mode인 경우 runDriver 메서드를 통해 YARN 클러스터 내에서 드라이버를 실행하여 애플리케이션을 제어합니다. runDriver와 다르게 Client 모드에서는 runExcutorLauncher가 실행되며 드라이버가 클러스터 외부에서 실행됩니다. 이 경우 runExecutorLauncher는 YARN에서 executor만 실행합니다.

```scala
if (isClusterMode) {
  runDriver()
} else {
  runExecutorLauncher()
}
```

#### [method] runDriver

userApplication을 별도의 스레드에서 실행합니다. 여기서 애플리케이션의 메인 클래스가 시작되며, 이 클래스는 클러스터에서 실행되는 드라이버 역할을 수행합니다. startUserApplication 메소드 내부에서 애플리케이션은 독립적인 스레드에서 실행되며, 이후 SparkContext가 초기화될 때까지 대기합니다.

```scala
userClassThread = startUserApplication()
```

userApplication에서 SparkContext가 초기화될 때까지 대기합니다. Spark 애플리케이션이 실행될 때, SparkContext는 클러스터에서 작업을 생성, 분해, 전달하는 객체입니다. executor를 생성하기전에 SparkContext가 정상적으로 설정되었는지 확인합니다.

```scala
val sc = ThreadUtils.awaitResult(sparkContextPromise.future,
  Duration(totalWaitTime, TimeUnit.MILLISECONDS))
if (sc != null) {
  val rpcEnv = sc.env.rpcEnv
}
```

Application Master를 YARN의 ResourceManager에 등록하여, 자원 할당 및 상태를 관리할 수 있게 합니다. 

```scala
registerAM(host, port, userConf, sc.ui.map(_.webUrl), appAttemptId)
```

executor의 상태 정보를 얻기 위해 driver와 SchedulerBackend 간의 통신을 설정합니다. 이후 클러스터 리소스를 할당하는 Allocator를 생성합니다. Allocator는 executor container 리소스를 관리하며 실행될 class를 관리합니다.

```scala
val driverRef = rpcEnv.setupEndpointRef(
  RpcAddress(host, port),
  YarnSchedulerBackend.ENDPOINT_NAME)
createAllocator(driverRef, userConf, rpcEnv, appAttemptId, distCacheConf())
```

### [class] YarnAllocator

ApplicationMaster class에서 createAllocator 메서드 호출이후 컨테이너 자원 할당을 위해 YarnAllocator가 호출됩니다. **YarnAllocator는 ResourceManager에 executor container를 요청하고 locality(호스트, 랙, ANYHOST)순으로 사용대기열에 올린다음 executorRunnable를 통해 exeuctor를 실행시킵니다.**

```scala
class YarnAllocator(
    driverUrl: String,
    driverRef: RpcEndpointRef,
    conf: YarnConfiguration,
    sparkConf: SparkConf,
    amClient: AMRMClient[ContainerRequest]) extends Logging {
  def allocateResources(): Unit = synchronized {...}
  def handleAllocatedContainers(allocatedContainers: Seq[Container]): Unit = {...}
  def runAllocatedContainers(containersToUse: ArrayBuffer[Container]): Unit = synchronized {...}
}
```

#### [method] allocateResource

ApplicationMaster 클래스에서 createAllocator 메서드 호출 이후 allocateResources가 실행됩니다. YARN ResourceManager에 자원 할당 요청을 보냅니다. allocateResponse는 YARN에서 현재 할당된 컨테이너와 사용 가능한 클러스터 자원 정보를 담고 있습니다.

```scala
val allocateResponse = amClient.allocate(progressIndicator)
```

allocateResponse를 통해 executor를 실행할 컨테이너들을 받은후 handleAllocatedContainers 메서드를 통해 할당된 컨테이너를 순회하며, 각 컨테이너에 Spark executor를 배포하여 애플리케이션이 클러스터에서 작업을 수행할 수 있게 합니다. 이 과정에서 Yarn NodeManager와 소통합니다.

```scala
val allocatedContainers = allocateResponse.getAllocatedContainers()
handleAllocatedContainers(allocatedContainers.asScala.toSeq)
```

#### [method] handleAllocatedContainers

이 리스트는 실제 사용할 컨테이너를 저장하는 곳입니다. 할당된 컨테이너 중에서 locality(호스트, 랙, ANYHOST) 기반으로 매칭후 애플리케이션에서 필요하다고 판단되는 컨테이너를 여기에 추가하게 됩니다.

```scala
val containersToUse = new ArrayBuffer[Container](allocatedContainers.size)
```

할당된 컨테이너와 요청된 호스트가 일치되는 container들 확안하는 작업입니다.
matchContainerToRequest 메서드는 컨테이너의 호스트가 애플리케이션이 요청한 호스트와 일치하는지 확인합니다. 일치하는 컨테이너는 위에서 선언된 containersToUse에 추가되고, 일치하지 않는 컨테이너는 remainingAfterHostMatches에 저장되어 이후 랙이 매칭되는지 확인합니다.

```scala
val remainingAfterHostMatches = new ArrayBuffer[Container]
for (allocatedContainer <- allocatedContainers) {
  matchContainerToRequest(allocatedContainer, allocatedContainer.getNodeId.getHost,
    containersToUse, remainingAfterHostMatches)
}
```

호스트에 맞지 않는 컨테이너에 대해 랙 매칭을 시도하는 코드입니다. resolver.resolve로 컨테이너가 있는 랙을 확인후 matchContainerToRequest()로 랙과 일치하는지 확인하고, 일치하는 컨테이너는 containersToUse에 추가됩니다. 일치하지 않는 경우 remainingAfterRackMatches에 저장됩니다.
YARN의 RackResolver가 thread interruption을 무시하기 때문에 메인 thread가 AM interruption을 무시하지 않도록 rack matching을 별도의 스레드에서 수행후 완료될때까지 대기합니다.

```scala
val remainingAfterRackMatches = new ArrayBuffer[Container]
if (remainingAfterHostMatches.nonEmpty) {
  val thread = new Thread("spark-rack-resolver") {
    override def run(): Unit = {
      for (allocatedContainer <- remainingAfterHostMatches) {
        val rack = resolver.resolve(allocatedContainer.getNodeId.getHost)
        matchContainerToRequest(allocatedContainer, rack, containersToUse,
          remainingAfterRackMatches)
      }
    }
  }
  thread.start()
  thread.join()
}
```

호스트와 랙에 모두 맞지 않는 컨테이너를 처리합니다. ANY_HOST라는 값을 사용하여, 특정 호스트나 랙에 상관없이 사용할 수 있는 컨테이너로 간주합니다. matchContainerToRequest 메서드를 통해 만약 일치하는 컨테이너가 없으면 remainingAfterOffRackMatches에 저장됩니다. 이후 locality 기반으로 매칭되지 않은 불필요한 컨테이너들은 YARN ResourceManager에 반환됩니다.

```scala
val remainingAfterOffRackMatches = new ArrayBuffer[Container]
for (allocatedContainer <- remainingAfterRackMatches) {
  matchContainerToRequest(allocatedContainer, ANY_HOST, containersToUse,
    remainingAfterOffRackMatches)
}
```

호스트, 랙, 오프랙 순 매칭을 통해 containerToUse에 저장된 컨테이너들에 Spark executor를 배치하는 메서드 입니다. runAllocatedContainers 메서드는 이 컨테이너들에서 실제로 executor를 실행하는 역할을 합니다.

```scala
runAllocatedContainers(containersToUse)
```

#### [method] runAllocatedContainers

executor 실행을 위해 ExecutorRunnable을 사용하여, 각 컨테이너에서 Spark executor를 시작합니다. ExecutorRunnable은 컨테이너 내에서 실행될 executor를 위한 자원(메모리, CPU)을 설정한후 nodemanager를 통해 executor를 실행하는 역할을 합니다. executor id와 target 갯수를 모니터링도 합니다. launcherPool 스레드풀을 이용해서 비동기적으로 여러 컨테이너에서 동시에 executor를 시작할 수 있게 해줍니다.

```scala
for (container <- containersToUse) {
  launcherPool.execute(() => {
    try {
      new ExecutorRunnable(
        Some(container),
        sparkConf,
        driverUrl,
        executorId,
        containerMem,
        containerCores,
      ).run()
      updateInternalState(rpId, executorId, container)
    } catch {
      case e: Throwable =>
        launchingExecutorContainerIds.remove(containerId)
    }
  })
}
```

### [class] ExecutorRunnable

NodeManager Client를 생성하여 NodeManager와 통신할 수 있도록 준비합니다. 이후 컨테이너의 실행 환경 객체를 생성후 cluster manager에 맞게 실행 class를 생성후 nm에 전달하여 spark executor를 실행합니다.

```scala
class ExecutorRunnable(...) extends Logging {
  def run(): Unit = { ... }
  def prepareCommand(): List[String] = { ... }
  def startContainer(): java.util.Map[String, ByteBuffer] = { ... }
}
```

#### [method] run

NodeManager와의 통신 설정: NMClient를 생성하여 NodeManager와 통신할 수 있도록 준비합니다.
startContainer() 메서드를 통해 컨테이너의 환경, 리소스를 설정후 실제로 Spark executor를 실행합니다

```scala
nmClient = NMClient.createNMClient()
nmClient.init(conf)
nmClient.start()
startContainer()
```

#### [method] startContainer

ContainerLaunchContext는 컨테이너가 실행될 때 필요한 정보(자원, 환경 변수, 명령어 등)를 담고 있는 객체입니다. 이 객체는 YARN NodeManager에게 전달되어 컨테이너의 실행 환경을 설정합니다.

```scala
val ctx = Records.newRecord(classOf[ContainerLaunchContext])
  .asInstanceOf[ContainerLaunchContext]
```

executor에서 실행할 때 필요한 JAR 파일, 라이브러리를 설정합니다. 
이후 prepareCommand 메서드를 통해 executor를 실행하기 위한 명령어를 준비합니다. yarn에서 실행시 container 클래스로 `org.apache.spark.executor.YarnCoarseGrainedExecutorBackend`를 지정합니다. ctx.setCommands()를 통해 ContainerLaunchContext에 설정됩니다.

```scala
ctx.setLocalResources(localResources.asJava)

val commands = prepareCommand()
ctx.setCommands(commands.asJava)
```

설정된 ContainerLaunchContext를 YARN NodeManager에 전달하여, 할당된 컨테이너에서 실제로 executor를 실행하는 작업을 수행합니다.

```scala
nmClient.startContainer(container.get, ctx)
```

## Executor

### [class] YarnCoarceGrainedExecutorBackend

YarnCoarceGrainedExecutorBackend는 executor container 애플리케이션의 시작 클래스입니다. 또한 YARN 환경에서 executor를 관리하는 CoarseGrainedExecutorBackend의 YARN 구현체입니다. YARN 컨테이너에서 필요한 속성을 얻고 executor를 관리하는 메서드가 추가되어있습니다.

```scala
class YarnCoarseGrainedExecutorBackend() extends CoarseGrainedExecutorBackend() with Logging {
  override def getUserClassPath
  override def extractAttributes
  override def extractLogUrls
}

object YarnCoarseGrainedExecutorBackend extends Logging {
  def main(args: Array[String]): Unit = { ... }
}
```

#### [method] main

AM에 의해 실행된후 executor의 진입점 method 입니다. createFn을 통해 YarnCoarseGrainedExecutorBackend 객체를 생성하고, 그 후 CoarseGrainedExecutorBackend.run() 메서드를 호출하여 executor를 시작합니다.

```scala
val createFn: (RpcEnv, CoarseGrainedExecutorBackend.Arguments, SparkEnv, ResourceProfile) =>
  CoarseGrainedExecutorBackend = { ... }
val backendArgs = CoarseGrainedExecutorBackend.parseArguments(args, ...)
CoarseGrainedExecutorBackend.run(backendArgs, createFn)
```


### [class] CoarceGrainedExecutorBackend

CoarseGrainedExecutorBackend는 Spark executor의 드라이버와의 통신 및 작업 처리를 관리하는 클래스입니다. executor와 드라이버 간의 작업 전송, 상태 관리등의 기능을 담당하며, executor에서 발생하는 작업을 처리하고 결과를 driver에 전송하기도 합니다.

```scala
class CoarseGrainedExecutorBackend() extends IsolatedThreadSafeRpcEndpoint with ExecutorBackend with Logging {
  override def onStart(): Unit = { ... }
  override def receive: PartialFunction[Any, Unit] = { ... }
}

object CoarseGrainedExecutorBackend extends Logging {
  def run(arguments: Arguments,
        backendCreateFn: (RpcEnv, Arguments, SparkEnv, ResourceProfile) => CoarseGrainedExecutorBackend): Unit = { ... }
}
```

#### [method] run

드라이버와의 RPC 연결을 설정합니다. setupEndpointRefByURI 메서드를 사용하여 엔드포인트를 설정하고, 이 과정을 통해 executor는 드라이버와 통신할 수 있게 됩니다.

```scala
val fetcher = RpcEnv.create("driverPropsFetcher", ... )
driver = fetcher.setupEndpointRefByURI(arguments.driverUrl)
```


backendCreateFn을 통해 CoarceGrainedExecutorBackend 객체를 executor 백엔드로 생성하고, 이를 RPC 엔드포인트로 등록합니다. 이로 인해 드라이버와의 통신이 활성화되며, 작업을 수신할 수 있게 됩니다.

```scala
val env = SparkEnv.createExecutorEnv(driverConf, ... )
val backend = backendCreateFn(env.rpcEnv, arguments, env, cfg.resourceProfile)
env.rpcEnv.setupEndpoint("Executor", backend)
```

#### [method] onStart

CoarceGrainedExecutorBackend 객체가 RPC 엔드포인트로 등록된후 onStart가 자동으로 호출되어, executor가 준비되었음을 드라이버에게 알립니다. asyncSetupEndpointRefByURI 메서드로 RPC 엔드포인트를 통해 드라이버와의 연결을 설정합니다. 이후 ref.ask(RegisterExecutor) 메서드를 통해 RegisterExecutor 메시지를 드라이버에게 보내서, executor가 드라이버에 성공적으로 등록될 수 있도록 요청합니다. 이후 드라이버가 executor를 등록하면 RegisteredExecutor 메시지를 스스로에게 보내 executor가 정상적으로 등록되었음을 알립니다.

```scala
rpcEnv.asyncSetupEndpointRefByURI(driverUrl).flatMap { ref =>
  driver = Some(ref)
  env.executorBackend = Option(this)
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

executor에서 rpc 요청을 받았을때 호출되는 메서드 입니다. RegisteredExecutor 메시지는 드라이버에 executor가 성공적으로 등록되었을 때 수신됩니다. executor가 등록되면 task를 수행하는 Executor 객체를 생성하고, 작업을 처리할 준비가 되었음을 알리는 LaunchedExecutor 메시지를 드라이버에게 전송합니다. executor 생성 중 오류가 발생하면, exitExecutor 메서드를 호출해 executor를 종료합니다.

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

LaunchTask 메시지는 드라이버가 executor에게 작업(Task)을 할당할 때 전송됩니다.
TaskDescription.decode 메서드로 할당된 TaskDescription을 디코딩하여 작업 설명을 추출합니다.
이후 executor.launchTask 메서드를 통해 디코딩된 taskDescription을 executor의 작업 실행 스레드에 전달하여 처리합니다. executor 객체가 생성되지 않은 상태에서 작업이 할당되면, executor를 종료합니다.

```scala
case LaunchTask(data) =>
  if (executor == null) {
    exitExecutor(1, "Received LaunchTask command but executor was null")
  } else {
    val taskDesc = TaskDescription.decode(data.value)
    logInfo(log"Got assigned task ${MDC(LogKeys.TASK_ID, taskDesc.taskId)}")
    executor.launchTask(this, taskDesc)
  }
```




# Conclusion

이번 분석을 통해 Spark-Submit 명령어로부터 시작하여 YARN ResourceManager와의 요청을 통해 Executor 컨테이너가 할당되고 ExecutorBackend가 시작되는 과정을 살펴보았습니다. 개인적으로 분석하면서 YarnAllocator가 사용할 container를 locality(호스트, 랙, ANYHOST) 우선순위 기반으로 선택해서 executor 서버를 실행하는 과정이 제일 흥미로웠습니다. 이후 driver의 TaskScheduler에서 task를 executor에 분배할때 locality(shuffle 최소화)를 사용하기 위한 사전작업이라고 생각합니다.

# Reference

---

[https://github.com/apache/spark/tree/master](https://github.com/apache/spark/tree/master)