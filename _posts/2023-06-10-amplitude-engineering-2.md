---
title:   "[Amplitude engineering 번역] - Scaling Analytics at Amplitude"
excerpt: "[Amplitude engineering 번역] - Scaling Analytics at Amplitude"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - Blog
  - Amplitude
  - TOC
last_modified_at: 2023-06-10T12:06:00+09:00
---

# Background

첫번째 레드시프트 성능개선에 이어서, 두번째 엠플리튜드의 초기 엔진이었던 amplitude wave에 관한 글을 정리해 보겠습니다.

분석 플랫폼을 만들기위해 어떠한 고민들을 했는지 경험할수 있는 좋은 기회인것 같습니다.

original post 글 주소입니다.

[Scaling Analytics at Amplitude](https://amplitude.engineering/scaling-analytics-at-amplitude-a95ee29342d3)

# Topic : Scaling Analytics at Amplitude

## Existing technologies

분석 플랫폼을 제공하기 위해서는 퍼포먼스, 빠른 응답 그리고 합리적인 가격이 필수적인 조건입니다. 아키텍쳐에 대해 설명하기전 현존하는 다른 기술들을 봐보죠. 모든 기술들은 크게 performance vs cost 관점에서 tradeoff가 있습니다.

high cost, high performance에서는 VoltDB와 MemSQL와 같은 in-memory db들이 있습니다. 이들은 좋은 퍼포먼스와 flexible queries들을 지원하죠. 하지만 보통 가격이 많이 비쌉니다. 빠르게 성장하는 앱이나 과도기에 보통 씁니다.

low cost, low performance로는 colum-store data warehouse가 있습니다. redshift가 대표적인데, 이 db는 유저 분석에 관한 패턴의 context가 부족합니다. 그리고 모두 disk에 넣기 때문에 좋은 성능을 내기는 어렵습니다.

그래서 우리의 aplitude wave architecture는 high performance와 low cost를 유지하기 위해 크게 두가지 
디자인 규칙을 만들었습니다. 

**pre-aggregation**과 **lambda architecture** 입니다.

## Amplitude wave architecture

### Pre-aggregation

---

보통 분석 시스템들은 쿼리를 계산하기위해 많은 data를 읽어와서 처리합니다. 
우리는 partial result를 pre-aggregate함으로써 final result를 더 빠른 속도로 낼수 있게 합니다.

행동학적 분석에 관해서 대부분의 질문들은 다음과 같은 공통점이 있습니다, 
**어떤 집단의 유저들이 특정한 행동을 하는가?**

그렇기 때문에 대부분의 질문들(segmentation, funnels, retention)은 여러 유저 집단의 교집합 계산하는것으로 표현될수 있습니다.

이러한 특성들 때문에, 행동학적 분석 관련된 query의 경우 유저집단을 특성별로 미리 계산해 놓음으로써 연산 과정을 획기적으로 줄일수 있습니다.

### Lambda Architecture

---

pre-aggregation은 다음과 같은 두가지 단점이 있습니다, 
mutable state(값이 real-time으로 update)와 
높은 저장 오버헤드(모든 pre-aggregate된 view table store) 입니다. 

다행히 lambda architecture의 idea를 통해 위 단점들을 해결할 단서를 찾았습니다.

만약 real-time update에 관심이 없다면, 우리는 매일 batch가 데이터들을 제공하면 됩니다. 정말 흥미로운점은 real-time update가 필요할때입니다. 만약 real-time update가 필요하다면 어떻게 될까요? 
batch view가 하루에 한번씩 제공되가 있다면, real-time으로는 하루동안 만들어 지고 있는 incremental한 데이터에 대해서만 처리를 하면됩니다. 그리고 하루가 지나고 batch view에 하루치 데이터가 새롭게 update가 된다면, speed layer에서 하루치 데이터를 제거하면 됩니다. 이로써 real-time update와 저장 오버헤드 issue 두가지 모두를 해결한것이죠.

정리하자면 람타 아키텍쳐는 쿼리를 layer들로부터 생성되는 view들로 분해하는 데이터 프로세스 모델입니다. layer로는 크게 두가지가 있는데  batch, serving layer(long interval update)가 있고, speed layer(real-time interval update)가 있습니다.
이구조의 가장 큰 장점은 지속적으로 growing하는 mutable state를 저장할 필요없이 real-time으로 처리할수 있다는 점입니다.

### Amplitude wave

---

대부분의 amplitude data 수집은 sdk를 통해 수집되고 대략 1분 이내로 server에 도착합니다.

speed layer(set database)는 보통 read와 write operation 위주의 작업이 필요합니다.
처음에는 postgresql을 사용해보았지만, scalability 측면에서 아쉬움이 있었습니다. membership-check를 위한 index, 확장되는 데이터를 감당하기에 disk의 확장이 힘들었습니다. postgresql은 general-purpose로는 좋은 db이지만, 저희의 요구사항에는 맞지 않았습니다.

그래서 저희는 redis와 비슷한 in-memory database를 개발하였습니다. in-memory operation 덕분에 매우 빠른 read와 write operation이 가능하였습니다. 또한 하루가 끝날때 데이터를 batch layer로 넘김으로써 memory의 cost도 아낄수 있었습니다.

batch layer로는 aws s3을 사용하였습니다. s3는 확장이 매우 용이하고, 다른 저장장치보다 가격이 낮습니다.
대부분의 분석 데이터들은 적재후 자주 활용되지 않는 “cold”한 특성때문에 가격 효율성을 우선순위로 선택하였습니다. 또한 batch layer는 write 보다는 read에 최적화가 되어있기때문에 에러 취약성이 낮고, 신뢰성이 높습니다.

마지막으로 query시, speed layer와 batch layer의 결과를 가져옵니다. 
lambda architecture로부터 파생된 분산 layer들은 낮은 가격, 그리고 view의 확장측면에서 많은 장점들이 있었습니다.