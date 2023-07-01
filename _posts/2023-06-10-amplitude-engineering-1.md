---
title:   "[Amplitude engineering 번역] - Optimizing Redshift Performance with Dynamic Schemas"
excerpt: "[Amplitude engineering 번역] - Optimizing Redshift Performance with Dynamic Schemas"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - Blog
  - Amplitude
last_modified_at: 2023-06-10T12:06:00+09:00
---
# Background

사내에서 분석 tool중 하나로 엠플리튜드를 쓰고있는데, 다양한 형태의 쿼리를 매우 빠른 응답시간으로 지원해주기 때문에 활발하게 쓰이고 있습니다. 
그래서 엠플리튜드의 분석 플랫폼이 어떻게 구성되어있고 블로그에 어떤 글들이 올라오는지 궁금해져서 내용을 요약해서 정리해보기로 했습니다.

첫 글로는 redshift 퍼포먼스 개선에 관한 글입니다.

분석 플랫폼을 만드는 엔지니어라면 각 문제를 어떻게 개선했을지 생각해보면서 읽으니까 재미있었습니다.

original post 글 주소입니다.

[Optimizing Redshift Performance with Dynamic Schemas](https://amplitude.engineering/optimizing-redshift-performance-with-dynamic-schemas-8acd3682f1ce)

# Topic : Optimizing Redshift Performance with Dynamic Schemas

## Our legacy redshift schema

기존의 redshift cluster를 운영하던 방법을 설명합니다.

### One massive table per app

---

데이터들이 파티션 되지 않고 통합되어 적재 되었기때문에,
한번 쿼리할때마다 관계 없는 이벤트들도 scan하여야 했습니다.

### Storing perperties in JSON strings

---

모든 user_properties, event_properties가 json string에 통합되어 저장되어 있었습니다. 유연성이 높기때문에 json 형태로 보유하고 있었습니다.
특정 properties 필요할때 `json_extract_path_text` 함수를 통해 json string전체를 다 읽어오고, 
cpu intensive 한 작업인 parsing 까지 진행했어야 했습니다.

### Multi-tenant clusters

---

여러 유저가 단일 cluster resource 공유했기때문에 서로의 사용성에 영향을 줄수 있었습니다.
만약 특정 쿼리가 수많은 row를 읽어야한다면 다른 쿼리들도 느려질것입니다.

## Introducing: Dynamic Dedicated Redshift

### Breaking out individual tables for each event type

---

모든 데이터들은 event type별로 partitioned 되어 있습니다.

특정 event type의 데이터를 탐색한다면, 이제 필요한 데이터만 scan하게 되었습니다.

### Dynamic schemas

---

각 event type 별로 분리하면서 dynamic schema가 가능하게 되었습니다.

각 event properties 별로 개별적인 column들을 가지게 되어서 `json_extract_path_text` 의 호출도 줄이게 되었습니다. 어떻게 event type 별로 dynamic한 schema를 가질수 있을까요?

우리는 일단 amazon s3에 json형태로 raw data들을 받습니다. 그리고 다음과 같은 transform 과정을 거칩니다
raw data들을 읽어서 각 event type file들을 생성하고, 각 event type file들을 읽고 schema file들을 생성합니다

이 transform 과정이 끝나고 redshift에 load하는 과정을 거치는데요, 
방금 생성했던 schema file들을 scan하면서 현재 redshift의 schema를 update 해야하는지 판별합니다. 
만약 update 해야 한다면, redshift의 schema들을 변경후 load 합니다.

### Dedicated

---

각 고객들은 독립적인 redshift cluster 를 할당받습니다. 
이뜻은 고객들간의 쿼리 성능이 서로 영향을 끼치지 않는다는 뜻입니다.

## Redshift Lessons Learned

### 1. Redshift로 효율적인 로딩을 위해 파일을 optimize 하라

---

레드시프트의 각 load job, commit statement에는 많은 오버헤드가 존재하기 때문에 작은 파일들을 큰 파일로 병합하는것이 더 효율적입니다. 개별의 작은 파일들 보다는 1MB의 파일 사이즈가 훨씬 빠른것을 발견했고, 일반적으로 1MB 부터 1GB사이의 파일사이즈를 로드합니다.

파일사이즈 또한 중요합니다. slice는 redshift에서 가장 작은 연산 단위이다. 각 node별 여러개의 slice들이 위치해있다.  resdhift의 copy를 사용할때 slice 갯수의 배수로 file들을 load하는것이 자원을 최대한 효율적으로 활용하는 것입니다.

### 2. 단일 transaction에 최대한 많은 command 를 통합하라

---

레드시프트에서 많은 병렬화를 지원하지만, 단일 commit queue가 존재하고, 그말은 transaction commit은 시퀀셜하게 일어난다는 뜻이다.

시퀀셜하고 연산작업이 많기 때문에 commit은 일반적으로 큰 병목현상입니다. 해서 최대한 많은 command를 한 transaction commit안에 포함시켜야 합니다.

### 3. Redshift의 워크로드 매니저의 이점을 살려라

---

유저 그룹 및 쿼르 구룹별로 리소스를 분배하기 좋다. 
또한 적절한 메모리 리소스를 분배함으로써, load 또는 query의 지연을 방지할수 있습니다.

### 4. 컬럼별 압축 형식을 지정해서 압축 비율을 높여라

---

데이터 압축을 통해서 저장공간을 아낀다면 비용측면 및 읽는 속도에서 효율적입니다. 레드시프트는 각 column별로  다른 압축 방식을 지원주기 때문에 generic한 strategy를 쓰는것보다 각 데이터 컬럼의 특성에 맞는 압축방식을 써야합니다.