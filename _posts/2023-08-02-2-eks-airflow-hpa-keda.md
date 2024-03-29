---
title:   "[EKS + Airflow] - HPA vs KEDA 비교하기"
excerpt: "[EKS + Airflow] - HPA vs KEDA 비교하기"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - eks
  - airflow
  - HELM
last_modified_at: 2023-08-02T12:06:00+09:00
---

# Background

---

airflow를 eks에서 운영하면서 pod 및 node scaling에는 여러 option들이 있는데요, airflow 운영시 pod-scaling option으로 hpa scaling 또는 keda component를 이용한 event-driven scaling 이 좋을지 비교하는 시간을 가져보도록하겠습니다. 또한 keda를 사용할경우 airflow 운영환경에 맞는 scaling 지표는 무엇인지도 알아봅시다:)

일단 k8s cluster를 운영하면서 pod autoscaling에 자주 사용되는  hpa와 keda를 간단히 정리해봅시다.

**hpa**(horizontal-pod-autoscaling)는 deployment와 statefulset과 같은 워크로드 리소스를 수요에 맞게 자동으로 크기 조정을 하는 k8s object입니다. 쿠버네티스 api 자원 및 컨트롤러 형태로 구현되어있는데요, 컨트롤러는 평균 cpu 사용률, 메모리 사용률 등 관측된 메트릭들을 목표에 맞추기 위해 워크로드 리소스 크기를 조정합니다. 조정 알고리즘은 아래 contents에서 세부적으로 다루어 보겠습니다.

**keda**(kubernetes event-driven autoscaling)도 워크로드 리소스들을 scaling 할수 있는 component 입니다. hpa와 같은 component와 같이 일하며 hpa의 수정 없이 여러 이벤트 소스로부터 event-driven하게 scaling 기능을 확장할수 있습니다. 운영을 위해서 CRD(custom resource definition) 과 k8s metric server를 사용합니다.

# Contents

---

hpa와 keda의 scale out 과정을 관찰하기위해 다음과 같은 세팅을 하였습니다.

hourly로 4개의 dag가 실행되고 각 dag는 2개의 task를 가져 병렬 처리되는 task의 최대 갯수는 8개입니다. 
또한 dag 한개당 celery worker 1개를 점유하고 cpu 80%를 5분동안 사용하기 때문에 최적화된 celery worker의 갯수는 4개입니다.

운영환경에서 worker pod scaling 적합성 비교 기준은 크게 2가지로 선정하였습니다.
- 빠른 scale out 으로 task들의 실행 시작이 빠른가? (8개 task 동시 처리)
- 적정 worker pod resource를 활용하는가? (worker pod 4개 확장)

실험을 위해 hpa, keda 각각 scale out시 위 두 기준을 잘 만족하는지 확인해보겠습니다.


## HPA

### HPA 설치

---

**Metric server**

```bash
# installing metric-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl get deployment metrics-server -n kube-system
# NAME             READY   UP-TO-DATE   AVAILABLE   AGE
# metrics-server   1/1     1            1           31s

# setting horizontal pod autoscaling
kubectl autoscale -n airflow deployment my-release-worker \
    --cpu-percent=60 \
    --min=1 \
    --max=10

kubectl get hpa -n airflow
# NAME                REFERENCE                      TARGETS   MINPODS   MAXPODS   REPLICAS
# my-release-worker   Deployment/my-release-worker   1%/60%    1         10  

```

### HPA 성능 평가

---

grafana를 사용해서 hourly로 dag가 trigger 될때 worker pod의 갯수와 airflow의 running task 갯수를 확인해보겠습니다. 

celery worker pod의 갯수를 확인하기 위해 **worker_replicaset_number** 지표를 사용하였고 running task 를 관찰하기 위해 **airflow_pool_running_slots_default_pool** 지표를 활용하였습니다.

`worker_replicaset_number`

![2.png](https://raw.githubusercontent.com/chaneeh/chaneeh.github.io/master/img/eks-airflow-hpa-keda/2.png)

`airflow_pool_running_slots_default_pool`

![3.png](https://raw.githubusercontent.com/chaneeh/chaneeh.github.io/master/img/eks-airflow-hpa-keda/3.png)


`worker_replicaset_number` 가 계단식으로 증가하고 worker capacity가 증가함에 따라 `running_slots` 갯수도 같이 증가하는것을 확인할수 있습니다. 이제 앞서 말했던 기준인 task의 시작 시점과 최적 worker pod resource 측면해서 분석해보도록 하겠습니다.

**[task 시작 시간]** running slot graph를 보았을때 리소스 부족으로 인해 마지막는 task는 4분이 지난 이후에 running state가 되었습니다. 이는 worker가 최적의 갯수(4개)로 바로 scale out이 되지 않고 hpa 로직에 따라 순차적으로 증가하였기 때문입니다.

**[resource 활용]** dag 4개를 동시실행하는데 적절한 worker 갯수는 4개이지만 hpa의 worker_replicaset_number 는 2배인 8개 까지 scale out을 시켰습니다. 또한 실행되는 task가 없음에도 scale in이 느리게 진행되는것을 확인할수 있습니다.

scaling의 작동원리를 살펴보자면, hpa의 targetAverageUtilization가 cpu-percent로 지정되면 scaling 타겟 워크로드 리소스 pod들의 cpu-utilization 평균을 기반으로 worker 갯수를 산정하고 공식은 다음과 같습니다.

```bash
target_replicaset_num = ceil[current_replicaset_num * (current_metric / desired_metric)]
```

hpa controller가 수식을 통해 ‘desired cpu utilization 밑으로 부하를 분산시키기 위한 replicaset 갯수’를 산정한다고 볼수 있겠네요. 이번 실험의 경우  celery worker의 cpu utilization은 평균 80% 정도이고, desired_metric은 60%로 잡았기 때문에 [current_metric / desired_metric]은 80/60 ⇒ 1.33 입니다.
초반 worker 갯수가 1 ~3개 일때는 replicaset이 한개씩 늘어나지만, 그 이후에는 2개씩 늘어납니다. 

```sql
ceil[1 * 1.33] => 2
ceil[2 * 1.33] => 3
ceil[3 * 1.33] => 4
ceil[4 * 1.33] => 6
ceil[6 * 1.33] => 8
```

hpa의 로그를 관찰하면 아래와 같습니다.

`kubectl get hpa -n airflow -w`

```bash
NAME                REFERENCE                      TARGETS   MINPODS   MAXPODS   REPLICAS
my-release-worker   Deployment/my-release-worker   1%/60%    1         10        1 
my-release-worker   Deployment/my-release-worker   55%/60%   1         10        1 
my-release-worker   Deployment/my-release-worker   81%/60%   1         10        1 
my-release-worker   Deployment/my-release-worker   81%/60%   1         10        2 
my-release-worker   Deployment/my-release-worker   81%/60%   1         10        3 
my-release-worker   Deployment/my-release-worker   81%/60%   1         10        4 
my-release-worker   Deployment/my-release-worker   81%/60%   1         10        6 
my-release-worker   Deployment/my-release-worker   64%/60%   1         10        8
```

`kubectl describe hpa -n airflow`

```bash
Type    Reason             Age                From                       Message
----    ------             ----               ----                       -------
Normal  SuccessfulRescale  32m (x3 over 27h)  horizontal-pod-autoscaler  New size: 2; reason: cpu resource utilization (percentage of request) above target
Normal  SuccessfulRescale  30m (x3 over 27h)  horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target
Normal  SuccessfulRescale  29m (x2 over 27h)  horizontal-pod-autoscaler  New size: 4; reason: cpu resource utilization (percentage of request) above target
Normal  SuccessfulRescale  27m                horizontal-pod-autoscaler  New size: 6; reason: cpu resource utilization (percentage of request) above target
Normal  SuccessfulRescale  26m                horizontal-pod-autoscaler  New size: 8; reason: cpu resource utilization (percentage of request) above target
Normal  SuccessfulRescale  21m                horizontal-pod-autoscaler  New size: 5; reason: All metrics below target
Normal  SuccessfulRescale  16m (x2 over 62m)  horizontal-pod-autoscaler  New size: 4; reason: All metrics below target
Normal  SuccessfulRescale  16m (x3 over 27h)  horizontal-pod-autoscaler  New size: 2; reason: All metrics below target
Normal  SuccessfulRescale  11m (x3 over 27h)  horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```

위와같은 계산 방식때문에 초반에 4개의 replicaset으로 확장되기까지 시간이 걸렸고, 
4개로 늘어났음에도 cpu_utilization이 desired_state 보다 높았기 때문에 2배인 8개까지 확장된것임을 확인할수 있습니다.  airflow의 task를 효율적으로 실행하기에는 아쉬운점이 있습니다.

이번에는 keda의 scaling을 확인해 봅시다.

## KEDA

### KEDA 설치

---

eks에서 airflow 운영시 keda 설정은 helm chart의 worker 속성에서 설정 할수 있습니다.

**`prod_airflow_helm_chart_values.yaml`**

```yaml
executor: "CeleryExecutor"
workers:
  keda:
    enabled: True

    # Minimum number of workers created by keda
    minReplicaCount: 1

    # Maximum number of workers created by keda
    maxReplicaCount: 10
		...
```

keda의 ScaledObject CRD는 외부 event source로부터 어떻게 target application을 scaling 할것인지 define합니다. 해당 trigger 로직과 scaling 결과는 아래에서 더 자세히 분석해보겠습니다.

**`worker-kedaautoscaler.yaml`**

```sql
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
	...
spec:
	...
  triggers:
    - type: postgresql
      metadata:
        targetQueryValue: "1"
        connectionFromEnv: AIRFLOW_CONN_AIRFLOW_DB
        query: >-
          SELECT ceil(COUNT(*)::decimal / {{ .Values.config.celery.worker_concurrency }})
          FROM task_instance
          WHERE (state='running' OR state='queued')
          {{- if eq .Values.executor "CeleryKubernetesExecutor" }}
          AND queue != '{{ .Values.config.celery_kubernetes_executor.kubernetes_queue }}'
          {{- end }}
```

### KEDA 성능평가

---

keda 또한 grafana를 사용해서 worker pod scaling을 분석해보았습니다.


`worker_replicaset_number`

![4.png](https://raw.githubusercontent.com/chaneeh/chaneeh.github.io/master/img/eks-airflow-hpa-keda/4.png)


`airflow_pool_running_slots_default_pool`

![5.png](https://raw.githubusercontent.com/chaneeh/chaneeh.github.io/master/img/eks-airflow-hpa-keda/5.png)


**[task 시작 시간]** worker pod가 1분 이내로 빠르게 scale out이 되어 8개 task 모두 1분뒤 병렬처리 되는것을 확인할수 있습니다.

**[resource 활용]** 최적의 replicaset number인 4개까지만 scale out을 하였습니다.

두가지 기준모두 keda의 정확한 scaling 덕분에 hpa보다 더 나은 모습을 보여주었습니다.

keda가 정확한 scaling갯수를 산정할수 있었던 이유는 targetvalue를 산정하는 로직의 차이때문인데요, airflow는 task 실행시 scheduler가 task_instance를 생성후 task_queue에 enqueue를 합니다. 그리고 keda는 해당 task_instance의 갯수를 기반으로 필요한 replicaset 갯수를 산정합니다. pod의 resource 기반이 아니라 airflow 내부에서 실행되는 event들을 기반으로 scaling 하는것입니다. 

```sql
SELECT ceil(COUNT(*)::decimal / {{ .Values.config.celery.worker_concurrency }})
FROM task_instance
WHERE (state='running' OR state='queued')
{{- if eq .Values.executor "CeleryKubernetesExecutor" }}
AND queue != '{{ .Values.config.celery_kubernetes_executor.kubernetes_queue }}'
{{- end }}
```

초반에 task_instance 8개가 enqueue되면 celery worker의 `minreplicaset`은 1개 이기 때문에 2개는 ‘running’ state, 6개는 ‘queued’ state 가 됩니다. worker_concurrency는 2 이기 때문에 keda에서 선정한 desiredstate는 4(= 8/2)가 됩니다. 이러한 airflow의 event 기반 scaling이 keda가 바로 적절한 worker 갯수인 4개를 target num로 산정할수 있는 이유입니다. hpa의 로그를 확인하면 다음과 같습니다.

`**kubectl get hpa -n airflow -w**`

```bash
NAME                         REFERENCE                      TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-my-release-worker   Deployment/my-release-worker   0/1 (avg)   1         10        1          45m
keda-hpa-my-release-worker   Deployment/my-release-worker   4/1 (avg)   1         10        1          45m
keda-hpa-my-release-worker   Deployment/my-release-worker   1/1 (avg)   1         10        4          45m
```

`**kubectl describe hpa -n airflow**`

```bash
Type            Status  Reason            Message
----            ------  ------            -------
AbleToScale     True    ReadyForNewScale  recommended size matches current size
ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from external metric postgresql-postgresql---airflow_user-airflow_pass@prod-airflow-cjlto4d4gcnr-ap-northeast-2-rds-amazonaws-com-5432-airflow_db(&LabelSelector{MatchLabels:map[string]string{scaledObjectName: my-release-worker,},MatchExpressions:[]LabelSelectorRequirement{},})
ScalingLimited  True    TooFewReplicas    the desired replica count is less than the minimum replica count
Events:
Type    Reason             Age   From                       Message
----    ------             ----  ----                       -------
Normal  SuccessfulRescale  26m   horizontal-pod-autoscaler  New size: 4; reason: external metric postgresql-postgresql---airflow_user-airflow_pass@prod-airflow-cjlto4d4gcnr-ap-northeast-2-rds-amazonaws-com-5432-airflow_db(&LabelSelector{MatchLabels:map[string]string{scaledObjectName: my-release-worker,},MatchExpressions:[]LabelSelectorRequirement{},}) above target
Normal  SuccessfulRescale  18m   horizontal-pod-autoscaler  New size: 3; reason: All metrics below target
Normal  SuccessfulRescale  17m   horizontal-pod-autoscaler  New size: 2; reason: All metrics below target
Normal  SuccessfulRescale  16m   horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```

# Conclusion

---

hpa와 keda를 각 기준별로 비교해보았고, task의 대기/실행시간 및 자원의 효율적 사용 측면에서 keda가 더 효율적인 모습을 보여준것을 확인할수 있었습니다. 이는 airflow의 내부 동작원리와 사용목적(scheduling)으로 인해, scaling의 target value를 worker의 resource 보다는 실행해야하는 task와 slot의 갯수를 통해 더 정확하고 빠르게 산정할수 있었기 때문입니다. 

또한 이후 백엔드 infra 구축시 서비스의 구성요소, 목적에 따라 scaling metric 전략을 다르게 해야한다는것도 배울수 있었습니다.

# Reference

---

**[k8s-docs : Horizontal Pod Autoscaling](https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale/)**

[**https://keda.sh/**](https://keda.sh/)