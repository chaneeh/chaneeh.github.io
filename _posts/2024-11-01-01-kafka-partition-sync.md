---
title:   "Kafka의 데이터 일관성을 유지하는 리더-팔로워 동기화 과정"
excerpt: "Kafka의 데이터 일관성을 유지하는 리더-팔로워 동기화 과정"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - kafka
last_modified_at: 2024-11-01T:12:30+09:00
---

# Motivation

Kafka는 대규모 데이터 스트리밍 환경에서 신뢰성과 일관성을 제공하는 분산 메시지 브로커입니다. 
특히, **리더-팔로워 구조**와 **ISR(In-Sync Replica)** 관리 메커니즘을 통해 클러스터 장애 복구와 데이터 안정성을 보장합니다. 

이전 글에서는 Kafka 클러스터 내 **topic metadata**가 생성되고 Kraft를 통해 각 브로커에 전달되어 커밋되는 과정을다뤘습니다. 
이번 글에서는 **리더와 팔로워 파티션의 데이터 동기화와 High Watermark(HWM) 관리**를 통해 실제 데이터 일관성을 어떻게 유지하는지 분석하고자 합니다.
특히, 리더와 팔로워 간의 **Fetch 요청 처리, 로그 동기화, HWM 갱신**이 어떻게 이루어지는지를 중심으로 분석합니다. 

이번 글이 Kafka 클러스터 내부에서 벌어지는 변화를 깊이 이해하는 데 도움이 되기를 기대합니다. 분석에 사용한 소스코드로 kafka version 3.9.0을 사용하였습니다.

# Contents

## 1. Follower Partition의 로그 복제 과정

### [class] ReplicaManager

ReplicaManager는 Kafka 브로커에서 파티션의 생성, 상태 업데이트를 관리하는 클래스입니다. 
팔로워 파티션의 초기화와 동기화를 담당하며, Fetcher 스레드 생성 요청을 통해 데이터를 가져오는 과정을 시작합니다.

#### [method] applyLocalFollowersDelta

이 메서드는 topic metadata가 커밋된 이후 호출되며, 파티션별 상태를 설정하고 Fetcher 스레드가 데이터를 가져오도록 준비합니다.
Topic metadata를 기반으로 파티션 상태를 생성하며 리더 변경 여부를 감지하여 Fetcher 스레드 시작을 ReplicaFetcherManager에게 요청합니다.

```scala
def applyLocalFollowersDelta(changedPartitions: mutable.Set[Partition], newImage: MetadataImage, delta: TopicsDelta,
  localFollowers: mutable.Map[TopicPartition, LocalReplicaChanges.PartitionInfo]): Unit = {  
  localFollowers.foreachEntry { (tp, info) =>
    getOrCreatePartition(tp, delta, info.topicId).foreach { case (partition, isNew) =>
      val isNewLeaderEpoch = partition.makeFollower(state, Some(info.topicId))
      if (isNewLeaderEpoch) {
        partitionsToStartFetching.put(tp, partition)
      }
    }
  }

  if (partitionsToStartFetching.nonEmpty) {
    val partitionAndOffsets = new mutable.HashMap[TopicPartition, InitialFetchState]
    partitionsToStartFetching.foreachEntry { (topicPartition, partition) =>
        val nodeOpt = partition.leaderReplicaIdOpt
        partitionAndOffsets.put(topicPartition, InitialFetchState(new BrokerEndPoint(nodeOpt.id, nodeOpt.host, nodeOpt.port)))
    }
    replicaFetcherManager.addFetcherForPartitions(partitionAndOffsets)
  }
}
```

### [class] ReplicaFetcherManager

ReplicaFetcherManager는 리더로부터 데이터를 가져오는 Fetcher 스레드를 생성 및 관리하는 클래스입니다.
브로커별 Fetcher 스레드를 관리하여, 리더-팔로워 파티션간 데이터 복제를 관리합니다.

#### [method] addFetcherForPartitions

각 브로커의 파티션에 적합한 Fetcher 스레드를 브로커 별로 그룹화하여 관리합니다.
리더 정보를 바탕으로 Fetcher 스레드를 생성 또는 재활용하고, 각 Fetcher 스레드에 파티션을 등록하여 start method를 통해 작업을 시작을 요청합니다.

```scala
def addFetcherForPartitions(partitionAndOffsets: Map[TopicPartition, InitialFetchState]): Unit = {
  val partitionsPerFetcher = partitionAndOffsets.groupBy { case (tp, state) =>
    BrokerAndFetcherId(state.leader, getFetcherId(tp))
  }

  def addAndStartFetcherThread(brokerAndFetcherId: BrokerAndFetcherId, brokerIdAndFetcherId: BrokerIdAndFetcherId): T = {
    val fetcherThread = createFetcherThread(brokerAndFetcherId.fetcherId, brokerAndFetcherId.broker)
    fetcherThreadMap.put(brokerIdAndFetcherId, fetcherThread)
    fetcherThread.start()
    fetcherThread
  }

  for ((brokerAndFetcherId, initialFetchOffsets) <- partitionsPerFetcher) {
    val fetcherThread = fetcherThreadMap.get(brokerIdAndFetcherId) match {
      case Some(currentFetcherThread)
        currentFetcherThread
      case None =>
        addAndStartFetcherThread(brokerAndFetcherId, brokerIdAndFetcherId)
    }
    addPartitionsToFetcherThread(fetcherThread, initialFetchOffsets)
  }
}
```


### [class] ReplicaFetcherThread

ReplicaFetcherThread는 팔로워 브로커에서 Fetch 요청을 리더로 전송하고, 리더로부터 데이터를 받아오는 스레드입니다. 
Fetch된 데이터를 처리하고 팔로워의 로컬 로그를 업데이트합니다.

#### [method] processFetchRequest

이 메서드는 thread start 이후 doWork method에 의해 지속적으로 호출되어 fetch를 진행합니다.
리더에게 fetch 요청을 한후, 수신한 데이터들을 각 파티션별로 processPartitionData를 통해 처리합니다.

```scala
def processFetchRequest(
  sessionPartitions: util.Map[TopicPartition, FetchRequest.PartitionData], 
  fetchRequest: FetchRequest.Builder): Unit = {
  responseData = leader.fetch(fetchRequest)

  if (responseData.nonEmpty) {
    responseData.foreachEntry { (topicPartition, partitionData) =>
      val logAppendInfoOpt = processPartitionData(topicPartition, currentFetchState.fetchOffset, partitionData)
      ...
    }
  }
}
```

#### [method] processPartitionData

이 메서드는 파티션별로 데이터를 처리하며, 로그 동기화와 HWM 갱신을 통해 팔로워와 리더 간 일관성을 유지합니다.
리더로부터 받은 데이터를 로컬 로그(UnifiedLog)에 추가하고, 전달받은 HWM 정보를 기반으로 팔로워 파티션의 HWM을 갱신합니다.

이제 리더가 follower partition들의 fetch request 정보를 기반으로 어떻게 HWM을 update 하는지 알아보겠습니다.

```scala
def processPartitionData(topicPartition: TopicPartition, fetchOffset: Long, partitionData: FetchData): Option[LogAppendInfo] = {
  val logAppendInfo = partition.appendRecordsToFollowerOrFutureReplica(records, isFuture = false)
  log.maybeUpdateHighWatermark(partitionData.highWatermark).foreach { newHighWatermark =>
    partitionsWithNewHighWatermark += topicPartition
  }
}
```


## 2. Leader Partition의 HWM Update 과정

### [class] Partition

Partition 클래스는 Kafka의 각 파티션에서 데이터를 저장하고, Fetch 요청을 처리하며, 
리더 파티션의 경우 팔로워 파티션의 FetchRequest를 기반으로 ISR 관리와 HWM 업데이트를 수행합니다. 

#### [method] fetchRecords

리더가 FetchRequest는 요청을 처리할때 호출되는 method 이며, FetchRequest는 consumer와 팔로워 파티션 모두 요청이 가능합니다.
consumer와 팔로워 파티션 요청 모두 readFromLocalLog를 통해 로그를 읽어 오지만,
팔로워 요청일 경우 updateFollowerFetchState를 통해 ISR 조건 충족 확인 및 HWM 업데이트를 진행합니다.

```scala
def fetchRecords(
  fetchParams: FetchParams, 
  fetchPartitionData: FetchRequest.PartitionData, 
  updateFetchState: Boolean): LogReadInfo = {
  if (fetchParams.isFromFollower) {
    val localLog = localLogWithEpochOrThrow(fetchPartitionData.currentLeaderEpoch)
    val logReadInfo = readFromLocalLog(localLog)

    updateFollowerFetchState(
      followerFetchOffsetMetadata = logReadInfo.fetchedData.fetchOffsetMetadata,
      fetchParams.replicaEpoch
    )
  } else {
    val localLog = localLogWithEpochOrThrow(fetchPartitionData.currentLeaderEpoch)
    val logReadInfo = readFromLocalLog(localLog)
  }
}
```


#### [method] updateFollowerFetchState

이 메서드는 Fetch 요청 이후 팔로워의 상태를 업데이트하고, 리더의 데이터 동기화를 위한 HWM을 갱신합니다.
maybeExpandIsr을 통해 ISR 조건을 만족하는 복제본을 추가하거나 제거하며 maybeIncrementLeaderHW을 통해 리더의 HWM을 갱신합니다.

```scala
def updateFollowerFetchState(
  replica: Replica,
  followerFetchOffsetMetadata: LogOffsetMetadata,
  leaderEndOffset: Long,
  brokerEpoch: Long): Unit = {

  maybeExpandIsr(replica)

  val leaderHWIncremented = if (prevFollowerEndOffset != replica.stateSnapshot.logEndOffset) {
    leaderLogIfLocal.exists(leaderLog => maybeIncrementLeaderHW(leaderLog, followerFetchTimeMs))
  }
}
```

#### [method] maybeIncrementLeaderHW

ISR 복제본, 또는 ISR에 추가될 가능성이 있는 복제본의 LEO(last end offset)을 비교하여 가장 낮은 LEO를 새 HWM로 설정합니다.
새롭게 계산된 HWM을 UnifiedLog로 전달하여 저장 및 관련 manager들에게 전달합니다.

```scala
private def maybeIncrementLeaderHW(leaderLog: UnifiedLog, currentTimeMs: Long = time.milliseconds): Boolean = {
  val leaderLogEndOffset = leaderLog.logEndOffsetMetadata
  var newHighWatermark = leaderLogEndOffset
  replicasMap.values.foreach { replica =>

    def shouldWaitForReplicaToJoinIsr: Boolean = {
      replicaState.isCaughtUp(leaderLogEndOffset.messageOffset, currentTimeMs, replicaLagTimeMaxMs) && isReplicaIsrEligible(replica.brokerId)
    }

    if (replicaState.logEndOffsetMetadata.messageOffset < newHighWatermark.messageOffset &&
        (partitionState.maximalIsr.contains(replica.brokerId) || shouldWaitForReplicaToJoinIsr)
    ) {
      newHighWatermark = replicaState.logEndOffsetMetadata
    }
  }

  leaderLog.maybeIncrementHighWatermark(newHighWatermark)
}
```

### [class] UnifiedLog

UnifiedLog는 Kafka의 로컬 로그 파일을 관리하는 클래스입니다. 
각 파티션의 데이터를 저장하고, HWM 및 트랜잭션 상태 변경을 관리합니다.

#### [method] updateHighWatermarkMetadata

HWM 값을 저장하고 ProducerStateManager와 LogOffsetsListener에 변경 사항을 전달합니다.
ProducerStateManager에 전달함으로써 트랜잭션 로그를 정리하고, 
LogOffsetsListener에 전달함으로써 Consumer 읽기 가능 범위를 조정합니다.

```scala
private def updateHighWatermarkMetadata(newHighWatermark: LogOffsetMetadata): Unit = {
  highWatermarkMetadata = newHighWatermark
  producerStateManager.onHighWatermarkUpdated(newHighWatermark.messageOffset)
  logOffsetsListener.onHighWatermarkUpdated(newHighWatermark.messageOffset)
}
```

위와 같은 과정을 리더 파티션에서 거치면서 새 HWM를 update하고,
이후 팔로워 파티션의 ReplicaFetcherThread.processPartitionData에서 리더의 HWM를 동기화 하는 과정을 거칩니다.




# Conclusion

Kafka의 리더와 팔로워 간 동기화 과정은 클러스터의 데이터 일관성을 보장하는데 주요한 메커니즘입니다.
리더 파티션은 Fetch 요청을 처리하며 ISR 상태를 관리하고 HWM를 업데이트하며, 팔로워 파티션은 리더로부터 데이터를 Fetch하여 로그를 추가하고 HWM을 갱신함으로써 동기화를 유지합니다.

특히, Partition.maybeIncrementLeaderHW 메서드는 ISR 복제본의 최소 LEO를 기반으로 리더의 HWM을 갱신하며, 이를 통해 Consumer의 데이터 접근 범위를 동적으로 조정합니다. 
이러한 설계는 Kafka가 대규모 스트리밍 환경에서도 높은 신뢰성과 확장성을 제공할 수 있는 기반이 됩니다.

# Reference

---

[https://github.com/apache/kafka](https://github.com/apache/kafka)