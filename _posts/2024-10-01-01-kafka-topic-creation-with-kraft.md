---
title:   "Kafka의 Raft 기반 Topic 생성과 Commit"
excerpt: "Kafka의 Raft 기반 Topic 생성과 Commit"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - kafka
last_modified_at: 2024-10-01T:12:30+09:00
---

# Motivation

Kafka는 높은 확장성과 안정성을 통해 대규모 데이터 스트리밍 환경에서 핵심적인 역할을 수행합니다. 특히, **토픽 생성 요청**은 단순한 메타데이터 추가 작업을 넘어, Raft 합의 프로토콜을 기반으로 리더와 팔로워 간의 **로그 복제**와 **commit 과정**을 포함합니다.

리더는 새로운 메타데이터 변경 사항을 생성하고, 이를 FetchRequest와 FetchResponse를 통해 팔로워들에게 전파합니다. 이후 과반수 이상의 노드에서 복제가 이루어지면 해당 변경 사항이 Commit됩니다. 이러한 복제와 Commit 과정은 Kafka가 분산 환경에서 데이터의 일관성과 안정성을 유지할수 있게 해줍니다.

이번 글에서는 Kafka의 Topic Creation 요청 처리 과정 중 **로그 복제와 commit 과정**에 초점을 맞춥니다. 이를 통해 Kafka의 내부 동작 원리를 이해하고, 안정적인 분산 시스템 설계에 대한 통찰을 제공하고자 합니다. 소스코드로 kafka version 3.9.0을 사용하였습니다.

# Contents

## **1. Topic Metadata 로그 복제 과정**

### [class] ControllerApis

클라이언트에서 들어온 Topic 생성 요청을 처리하여 `Controller`로 전달하는 class 입니다.
`ControllerApis`는 클라이언트와 `Controller` 사이의 API 계층으로, 클라이언트 요청을 파싱하고 처리 결과를 반환합니다.

#### [method] handleCreateTopics

이 메서드는 클라이언트로부터 들어온 Topic 생성 요청(`CreateTopicsRequest`)을 처리하고 `Controller`로 전달합니다.
Topic 생성 요청 데이터를 파싱후, `createTopics` 메서드에 context와 요청 데이터를 넘겨 `QuoromController`가 로그를 생성하도록 요청합니다.


```java
private def handleCreateTopics(request: RequestChannel.Request): CompletableFuture[Unit] = {
    val createTopicsRequest = request.body[CreateTopicsRequest]
    val future = createTopics(context, createTopicsRequest.data, ...)
    future.handle[Unit] { (result, exception) =>
        val response = new CreateTopicsResponse(result)
        requestHelper.sendResponseMaybeThrottleWithControllerQuota(..., request, response)
    }
}
```

### [class] ReplicationControlManager

Topic 및 Partition 메타데이터 로그를 생성하고, 이를 기반으로 복제본 배치를 결정하는 클래스 입니다. 

#### [method] createTopic

`ReplicationControlManager`는 Topic 및 Partition 관련 메타데이터를 생성하고, 이를 기록할 `TopicRecord`를 만듭니다.

`replicaPlacer`를 이용해 Partition과 Replica 배치를 생성합니다. 이후 `TopicRecord` 형태로 메타데이터 로그를 생성하고 `QuoromController`에 반환합니다.

`replicaPlacer`의 로직은 `StripedReplicaPlacer` class에서 확인할수 있는데, rack과 broker별로 최대한 고르게 분산하도록 우선순위를 두고 있습니다.

```java
ControllerResult<CreateTopicsResponseData> createTopic(
    ControllerRequestContext context,
    CreateTopicsRequestData request,
    Set<String> describable
) {
    // Partition 배치 계산
    TopicAssignment topicAssignment = clusterControl.replicaPlacer().place(new PlacementSpec(
        0, numPartitions, replicationFactor
    ), clusterDescriber);

    // TopicRecord 생성
    records.add(new ApiMessageAndVersion(new TopicRecord().setName(topic.name()).setTopicId(topicId), (short) 0));

    // QuorumController에게 result 반환
    return ControllerResult.atomicOf(records, data);
}
```

### [class] QuorumController

`ReplicationControlManager`가 생성한 메타데이터 로그를 `KafkaRaftClient`에 전달하여 Raft 프로토콜로 복제를 시작하는 클래스 입니다.

#### [method] appendWriteEvent

`ReplicationControlManager`에서 생성된 topic meatadata log를 작업 큐에 넣어 비동기로 처리하도록 합니다.

```java
<T> CompletableFuture<T> appendWriteEvent(
    String name,
    OptionalLong deadlineNs,
    ControllerWriteOperation<T> op,
    EnumSet<ControllerOperationFlag> flags
) {
    ControllerWriteEvent<T> event = new ControllerWriteEvent<>(name, op, flags);
    queue.append(event);
    return event.future();
}
```

#### [method] run

`appendWriteEvent`로 인해 작업 큐에서 비동기로 실행시, WriteEvent의 run method를 통해 생성된 topic metadata가 `KafkaRaftClient`에 append되고, 이후 follower의 fetch 요청에 응답할수 있는 환경이 됩니다.

```java
@Override
public void run() throws Exception {
    ControllerResult<T> result = op.generateRecordsAndResult();
    long offset = appendRecords(log, result, maxRecordsPerBatch, records -> {
        raftClient.prepareAppend(controllerEpoch, records);
        raftClient.schedulePreparedAppend();
    });
}
```


### [class] KafkaRaftClient

리더와 팔로워 간 로그 복제 및 commit 과정을 관리합니다.
`KafkaRaftClient`는 Raft 프로토콜의 구현체로, 리더는 로그를 전송하고 팔로워는 이를 수신 및 기록합니다.

#### [method] handleFetchRequest

팔로워 노드는 리더 노드에게 주기적으로 Fetch 요청을 보내고, 리더는 전송 준비가 완료된 metadata log들을 `handleFetchRequest` method 내에서 `FetchResponse`에 담아 메타데이터 로그를 전달합니다. 

```java
private CompletableFuture<FetchResponseData> handleFetchRequest(
    RaftRequest.Inbound requestMetadata, long currentTimeMs
) {
    FetchResponseData response = tryCompleteFetchRequest(...);
    return completedFuture(response);
}
```

#### [method] handleFetchResponse

팔로워는 리더의 `FetchResponse`를 처리하여 `UnifiedLog`에 기록하도록 전달하고, HWM를 기반으로 새로운 로그가 Commit 가능한지 확인합니다. 
과반수 이상의 팔로워들이 로그 복제가 완료되어야 HWM가 update되어 commit process가 진행 됩니다.

```java
private boolean handleFetchResponse(RaftResponse.Inbound responseMetadata, long currentTimeMs) {
    FetchResponseData response = (FetchResponseData) responseMetadata.data();
    Records records = FetchResponse.recordsOrFail(partitionResponse);
    if (records.sizeInBytes() > 0) {
      appendAsFollower(records); // metadata 로그 기록
    }
    updateFollowerHighWatermark(state, highWatermark); // HWM 기반 커밋 여부 확인
    return true;
}
```


#### [method] appendAsFollower

팔로워 노드는 리더로부터 받은 메타데이터 로그를 `UnifiedLog.appendAsFollower` 호출을 통해 disk에 기록합니다.
팔로워는 리더로부터 토픽 생성 메타데이터를 전달받고 복제한 상황이고, 아직 broker에게 전파되지는 않은 상황입니다. 이후 commit 과정을 통해 `MetadataCache`에 전파되고, 실제 broker에 적용이 됩니다.

```java
private void appendAsFollower(Records records) {
    LogAppendInfo info = log.appendAsFollower(records);
    if (quorum.isVoter() || followersAlwaysFlush) {
        log.flush(false);
    }
    partitionState.updateState();
}
```

## **2. 리더와 팔로워의 Commit 과정**

### [class] LeaderState
#### [method] maybeUpdateHighWatermark

리더는 팔로워들의 `FetchRequest` 요청에서 팔로워들의 정보들을 저장합니다. 이후 주기적으로 `LeaderState`를 update 함으로써 Raft 프로토콜을 통해 팔로워 노드들의 과반수(quorum) offset을 기반으로 HWM을 갱신합니다. 과반수 이상의 팔로워가 특정 offset에 도달하면 HWM을 갱신합니다.

```java
private boolean maybeUpdateHighWatermark() {
  // Find the largest offset which is replicated to a majority of replicas (the leader counts)
  ArrayList<ReplicaState> followersByDescendingFetchOffset = followersByDescendingFetchOffset()
      .collect(Collectors.toCollection(ArrayList::new));

  int indexOfHw = voterStates.size() / 2;
  Optional<LogOffsetMetadata> highWatermarkUpdateOpt = followersByDescendingFetchOffset.get(indexOfHw).endOffset;

  if (highWatermarkUpdateOpt.isPresent()) {
    // update HighWaterMark
  }
}
```

---

### [class] KafkaRaftClient
#### [method] updateFollowerHighWatermark

팔로워는 `FetchResponse`에서 리더의 HWM 정보를 확인하고, HWM이 변경 되었다면 해당 offset까지 로그를 Commit합니다. 리더의 HWM에 따라 로그를 Commit후 해당 정보를 `listenerprogress`를 update함으로써 `MetadataCache`에 반영 및 전파합니다.

```java
private void updateFollowerHighWatermark(
        FollowerState state,
        OptionalLong highWatermarkOpt
    ) {
    highWatermarkOpt.ifPresent(highWatermark -> {
            long newHighWatermark = Math.min(endOffset().offset(), highWatermark);
            if (state.updateHighWatermark(OptionalLong.of(newHighWatermark))) {
                logger.debug("Follower high watermark updated to {}", newHighWatermark);
                log.updateHighWatermark(new LogOffsetMetadata(newHighWatermark));
                updateListenersProgress(newHighWatermark);
            }
        });
}
```

### [class] BrokerMetadataPublisher

`controller`에서 broker로 메타데이터를 publish하는 class 입니다.

#### [method] onMetadataUpdate

최종적으로 커밋된 메타데이터 변경 사항이 Broker에 전달됩니다. 이 과정에서 `MetadataCache`를 갱신하고, 변경 사항을 topic과 partition, ISR을 관리하는 ReplicaManager에 전달합니다.

```scala
override def onMetadataUpdate(delta: MetadataDelta, newImage: MetadataImage, manifest: LoaderManifest): Unit = {
    metadataCache.setImage(newImage)
    Option(delta.topicsDelta()).foreach { topicsDelta =>
        replicaManager.applyDelta(topicsDelta, newImage)
    }
}
```


# Conclusion

Kafka에서 Topic 생성 요청은 단순히 메타데이터를 기록하는 것을 넘어, **Raft 프로토콜**을 통해 로그를 복제하고 commit을 통해 Broker에 반영되는 과정을 포함합니다. 이번 글에서는 **로그 복제 및 commit**에 초점을 맞춰 주요 클래스를 분석하였습니다. 

리더는 Topic 생성 메타데이터를 Raft 로그로 작성한 뒤 팔로워들에게 fetch response를 통해 전송합니다. 이후 과반수 이상 복제가 완료되었을때 High Watermark를 업데이트하고 팔로워는 이를 기반으로 Commit을 완료해 MetadataCache를 업데이트하고, Broker에 전달하여 최종적으로 클러스터 상태를 반영합니다. 이러한 구조로 kafka가 분산 환경에서도 확장성과 장애복원력을 가질수 있게 되었다고 생각합니다.

다음 글에서는 Broker가 커밋된 메타데이터를 실제로 활용하는 과정을 다룰 예정입니다.

# Reference

---

[https://github.com/apache/kafka](https://github.com/apache/kafka)