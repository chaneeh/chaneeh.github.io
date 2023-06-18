---
title:   "Amplitude engineering blog 정리 - 3"
excerpt: "Amplitude engineering blog 정리 - 3"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - Blog
  - Amplitude
  - TOC
last_modified_at: 2023-06-18T12:06:00+09:00
---

# Background

지난 amplitude wave에 이어서 3편으로는 다음 분석 엔진인 nova에 대해 설명한 글을 정리해보겠습니다.

original post 글 주소입니다.

**[Nova: The Architecture for Understanding User Behavior](https://amplitude.engineering/nova-the-architecture-for-understanding-user-behavior-aa79dc8e9ef3)**

# Topic - **Nova: The Architecture for Understanding User Behavior**

## Intro

우리가 지속적으로 분석 feature를 확장함으로써 data storage에 무엇이 필요한지에 대한 이해도가 올라갔습니다. wave로는 부족해졌고, 더 복잡한 분석쿼리 및 비용 효율적인 모델이 필요해졌습니다.

프로덕트를 지속적으로 개선하기 위해 두가지 목표가 생겼습니다. 첫번째는 복잡한 행동 분석을 가능하게 하는것이고, 두번째는 비용 효율적인 확장성입니다. 이 두가지 목표를 달성하기 위해 우리는 in-house column store인 Nova 를 개발하게 되었습니다.

### Limitations of pre-aggregation

기존의 pre-aggregation은 제한된 query 종류에 대해서만 효율적이었습니다. 하지만 compass 또는pathfinder와 같은 복잡한 분석 쿼리의 수요가 증가하면서  pre-aggregation보다 더 높은 **query flexibility** 가 필요해졌습니다. 기존의 pre-aggregate된 정보를 쓰지않는다면, pre-aggregation 절차는 필요가 없어질것입니다.

유저들에게 더 다양한 분석 옵션을 제공해주기 위해 자연스럽게 더 많은 flexibility가 지원되며 raw data에 접근할수 있는 in-house column store을 고려하게 되었습니다.

### Overview of column stores

기존의 row 기반 db는 single 데이터의 crud에 특화되어있는 장점이 있지만, column 기반 db는 데이터가 저장된 구조덕분에 많은 양의 데이터 빠르게 답하기에 최적화 되어있습니다. 또한 raw data에 접근할수 있기 때문에 pre-aggregate system보다 더 많은 쿼리 유연성을 제공합니다.

column 기반 db는 크게 service, commercial, 그리고 open-source 세가지 카테고리가 있습니다.
service에는 aws redshift, google bigquery가 있고, commercial에는 hp vertica, c-store가 있고, open-source에는 citus data와 druid가 있습니다.

### Advantages of an in-house column store

이렇게 기존에 개발된 column db들이 있지만, 우리는 다음과 같은 이유때문에 in-house column store를 새롭게 개발하였습니다.

1. 엠플리튜드는 지속적으로 분석 feature를 개발할것이기 때문에  쿼리 유연성은 매우 중요한 측면중 하나입니다. 우리는 간단한 집계부터 머신러닝 알고리즘까지 다루는것이 중요합니다.
2. 데이터가 immutable하기 때문에, 그 성질을 이용해서 performance를 개선할수 있습니다. 이후에 몇몇의 mutability가 있지만, 일반적으로는 update기능을 지원하지 않기때문에 많은 complexity를 줄일수 있었습니다.
3. 우리가 데이터를 수집할때, 짧은 시간 텀 이후에 쿼리가능하기만 하면 되기 때문에, 바로 사용가능할 필요는 없습니다. immutablity와 비슷하게 consistency를 줄이는것도 최적화에 도움이 되었습니다.
4. 마지막으로, 데이터가 저장되는 방식을 결정할수 있기때문에 저장 비용을 줄일수 있습니다.

이러한 장점들과 별개로, 새 query engine을 개발하기 위해서는 많은 basic reliablity와 scaling mechanism을 개발해야하는 단점이 있습니다. 다행이 aws ecosystem을 이용해서 개발에 드는 많은 비용과 시간을 아낄수 있었습니다. 우리의 main 경쟁력이 분석과 분산 시스템이란것은 감안하였을때, 새 engine을 만드는것은 충분히 가치있는 투자였습니다.

## Introducing Nova

### User-centric computational model in Nova

우리는 Nova를 user centric한 개념을 기반으로 한 실시간 map-reduce를 지원하는 분산형 columnar storage 입니다. 상기해보자면, 쿼리 유연성은 매우 중요한 feature이고, map-reduce는 많은 type의 연산을 지원하는 generic한 framework 입니다.

행동 분석은 일반적으로 유저별로 관련된 모든 이벤트들을 한번에 봅니다. 이를 위해서 분산 시스템에서는 유저들의 분산된 join이 중요하고, 그 말인 즉슨 user들 기반으로 paritioning되어서 data moving이 최소화 되어야한다는 뜻입니다. 그렇게 됨으로써 독립적인 워커들로부터 distinct한 user set들 연산이 병렬적으로 이루어질수 있습니다.

예를 들어, 유저들의 funnel별 conversion을 계산한다 하였을때, distinct한 user set들끼리 aggregate한 이후, 규모가 축소된 집계된 데이터들을 aggregate 함으로써 계산을 진행할수 있습니다.
자세히 보시면, data layer에서 leaf node로 shuffle 과정이 있는것을 보실수 있습니다. 유저 파티셔닝이 잘 되어있다면 shuffle 과정이 생략되서 map, reduce 과정이 없어져도 되지 않냐 생각하실수 있는데요. 저희는 특정 미래시점에서 두가지 distinct 한 유저가 사실 같은 유저임을 발견할수 있습니다. 데이터들은 immutable하다는 원칙이 있기 때문에 우리는 위와 같은 상황에서는 query-time mapping을 이용해서 문제를 해결하려 했고, 이는 data layer에서 완벽히 user별로 partitioning이 될수는 없어서 shuffle이 일어날수도 있음을 의미합니다. 하지만 그런 경우는 매우 적은 경우라서 해당 경우의 cost는 적습니다.

마지막으로 reduce 단계에서 유저 subset들에 대한 query 결과를 계산하였을때 aggregate layer로 전송됩니다. 이또한 데이터 크기가 작지 않기 때문에 우리는 계층적인 집계를 사용합니다.

### The underlying storage engine

우리는 labmda architecture를 또 채용하기로 하였습니다. 복기하자면, 람다 아키텍쳐는 real-time update의 복잡도와 비용을 localize하기위해 real-time ingestion과 batch-computed data를 분리시키는 디자인 패턴입니다. 이전 구조와 달라진 점은, layer 들이 pre-aggregate 하는데 쓰이는게 아닌, 최적화된 columnar format으로 변환하는데 쓰인다는것입니다.

또한 우리는 AWS s3를 지속성과 비용이라는 큰 장점 때문에 매우 활발하게 쓰고 있습니다. s3 덕분에 다른 시스템들을 도입할때 replication걱정 없이 cache로써 활용할수 있습니다.

우리는 성능 개선을 위해 데이터들을 최적화해서 저장합니다. 다루기 쉬운 immutable chunk들을 만들기 위해 user, timestamp 그리고 ingestion batch로 나눕니다. 각 chunk들에는 여러 column 기반 압축 기술들이 적용되었는데요, 예를들어 delta encoding timestamp가 있고, user id와 device type같은 여러 column들에는 dictionary encoding이 적용되었습니다. dictionary encoding은 value에 따라 event들을 저장했기때문에 압축뿐만 아니라 group by에도 효율적인 압축 방식입니다; hashmap lookup을 array lookup으로 바꿈으로써 cpu-bound 연산에 큰 개선을 보일수 있습니다.
columnar data format으로 일반적인 포멧을 썻고, 데이터가 클때는 LZ4 compression algorithm을 썼습니다.