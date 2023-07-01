---
title:   "[Amplitude engineering 번역] - Nova 2.0 : Re-architecting the Analytics Engine behind Amplitude"
excerpt: "[Amplitude engineering 번역] - Nova 2.0 : Re-architecting the Analytics Engine behind Amplitude"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - Blog
  - Amplitude
last_modified_at: 2023-06-18T12:06:00+09:00
---

# Background

이번 amplitude 번역 네번째 글은 nova 1.0을 개선한 nova 2.0 개발기입니다.


기존 분석 엔진인 nova를 개선하기 위해 로그 데이터 지역 특성에 따라 데이터 적재 및 탐색방식을 external file metadata system을 만듬으로써 개선하고 로그 데이터 조회 특성에 맞게 자주 사용되는 column 위주로 캐싱 전략도 바꾼점이 매우 흥미로웠습니다. 

이후 저도 데이터기반 백엔드 서비스를 구축할때 서비스의 데이터 생성 및 활용 특성을 고려하여 적재, 탐색, 조회 기반 인프라를 설계해야하는 교훈을 얻게 되었습니다.

original post 글 주소입니다.

**[Nova 2.0: Re-architecting the Analytics Engine behind Amplitude](https://amplitude.engineering/nova-2-0-re-architecting-the-analytics-engine-behind-amplitude-55568420dce7)**

# Topic - Nova 2.0 : Re-architecting the Analytics Engine behind Amplitude

고객 및 데이터 수가 증가하면서 low latency를 유지하는것은 우리에게 매우 중요한 feature가 되었습니다.
우리는 re-architect를 함으로써 P95 latency를 3배나 줄일수 있었고, 서버 비용도 10% 줄일수 있었습니다.

### What is Nova?

Nova란 엠플리튜드의 in-house columnar-based storage & analytics engine 입니다. 

### Why re-architect?

1. uncached된 쿼리들의 중간값들을 0.5초 이하의 시간이 걸렸지만, P95의 query들은 2.5초에서 8초 이상까지 증가하였습니다.
2. 우리는 지속적으로 새 분석 feature들을 개발하였습니다. 새로운 다양한 feature들은 복잡한 business logic과 데이터 acess 패턴때문에 nova에 추가적인 부하를 주었습니다.
3. 우리의 주요 특징중 하나는 유저들에게 쿼리 타임에 상호적인 데이터 변화를 주는 query-time transformation입니다. 파일들이 immutable 하기 때문에 이 기능을 지원하기 위해 추가적인 Complexity가 생겼습니다.
4. 서비스를 운용하면서 사용자들의 쿼리 패턴을 알게 되었습니다.

## Research and Investigation

시스템의 퍼포먼스를 분석하면서 우리는 다음과 같은 몇가지 발견들을 하였습니다.

### Fragmentation of Data into lots of small files

느린 쿼리들중 대부분은 디스크에서 raw data를 읽어오는것, 특히 작은 여러 파일들을 읽어오는데 많은 시간이 걸린다는 것을 발견하였습니다. 이는 amplitude의 특징에서 오는 특이한 현상인데요, 저희는 크게 두가지 timesatmp가 있습니다. event time과 server upload time인데요. client 쪽의 인터넷 연결 문제로 인해서 이 두가지 timestamp는 종종 큰차이가 벌어지곤 합니다. 한달 넘게 차이나는 경우도 지속적으로 발생합니다.

모든 분석 쿼리들은 time range filter이 있기 때문에 우리는 event time 기준으로 indexed 되어 있기를 원했습니다.

### Customers query only a few select columns

평균적으로 프로젝트별로 189개의 column들이 있지만 쿼리당 쓰이는 컬럼은 3 ~ 4개에 불과하였습니다. 또한 지난 30일동안 고객별 평균 쿼리한 컬럼갯수들은 16개에 불과하였습니다. 특정 데이터들은 모든 column들이 한 파일에 있었고, 90%의 데이터들은 local disk에 옮겨졌음에도 불구하고 쓰이지 않았습니다.

## Architectural Changes

### Introducing Dynamic time-ranges files to reduce P95 latency by 3x:

우리는 file들의 immutable을 기본 원리로 사용하고 있고, server에 데이터가 뒤늦게 들어 왔을때 event_time 의 subdirectory에 작을 파일들을 추가하는 방식으로 운용하고 잇었습니다. 이로인해 작은 파일들이 지속적으로 증가하였고, 읽는 횟수가 많아지는 문제점이 있었습니다. 이를 해결하기위해 우리는 더 큰 사이즈의 파일들을 만들었습니다.

기존에는 {weekly event time bucket} / {server upload date} 로 bucketing이 되어있어서 한달치 데이터를 읽고 싶다면 5개의 subdirectory를 읽어야 했습니다.

만약 우리가 파일당 더 많은 데이터를 수용하기 위해 정적인 weekly event time bucketing을 포기한다면, 우리는 dynamic한 time range를 가져야합니다. 이럴경우 우리는 파일별 시작 시간과 종료시간을 포함하는 metadata store를 가져야합니다. 그리고 우리는 쿼리 요청시 그 metadata store에 어떤 file들을 scan해야하는지 요청할수 있습니다. 이는 `file metadata store`로 추가하게 되었습니다. 예를 들면 다음과 같은 정보들을 담고 있습니다. file_id, event_start_time, event_end_time, server_upload_time, number_of_rows 등등 입니다.

우리는 dynamo db를 file metadata store로 사용하고 있습니다. 이 정보는 하루에 한번 바뀌기 때문에 우리는 metadata store results들을 query node들의 disk와 memory들에 많이 caching하고 있습니다.

dynamic한 file range를 구성한다고 하였을때 중요한것은 time range를 설정하는것입니다. 너무 많은 file 갯수들은 파일 읽는 횟수의 감소를 달성하지 못해서 파일을 opening & closing하는 오버헤드가 있을것이고, 너무 적은 file 갯수들은 event_time index의 장점을 못살려 적절한 scan filtering을 달성하지 못할것입니다. 그래서 우리는 다음과 같은 두가지 타입의 파일들을 원하였습니다.

1. 짧은 기간의 정보이면서 많은 양의 row를 가지고 있는 file입니다.
2. 넓은 기간의 정보이면서 적은 양의 row를 가지는 file 입니다. 넓은 기간을 포함하고 있는 file이기 때문에 자주 쿼리에 포함될것이지만, 양이 적기때문에 크게 부담스럽진 않습니다.

이렇게 파일 갯수들을 줄이면서 우리는 P95 latency를 크게 줄일수 있었습니다.

### Introducing a Column Metadata Store and reducing server costs by 20%:

위에서 언급한 file metadata store가 어느 파일을 읽을지 알려주면 Nova는 관련이 있는 파일들을 s3에서 가져옵니다. 파일의 형태는 이전 시리즈에서 작성했듯이, user, timestamp, 그리고 column들로 나뉘어 있으며 각각의 column들은 압축과 group by의 효율성을 위해 dictionry encoded 압축 방식으로 저장되어 있습니다.

metadata header에는 각 column들의 start와 end offeset 정보들이 담겨져 있습니다. 우리가 앞서 말한대로 대부분의 column 정보들은 거의 쓰이지 않습니다. 전체를 다운받는대신 우리는 관련있는 column 정보들만 다운받을것입니다. 

우리는 각 column들을 분리해서 s3 object 형태로 저장할수 있지만 s3는 Put의 횟수대로 과금을 하기 때문에 이는 비용을 늘릴것입니다. 대신에 우리는 s3의 partial object read capabilities를 사용함으로써 start와 end offset을 이용해 관련있는 정보들만 가져올것입니다. 이 use case를 충족하기 위해, 우리는 다른 distributed된 data store를 이용해 각 파일별로 column당 start와 end offset 정보를 저장하였고, 이를 column metadata store로 부릅니다. 

이제 우리는 관련있는 정보들만 disk에 저장하기 때문에 저장 비용이 많이 줄었습니다. 우리는 ec2에 s3 정보를 caching하고 있는데 비용이 많이 줄었습니다. instance 성능의 90를 감축함으로써 우리는 더 적절한 ec2 type을 선택할수 있게 되었습니다.