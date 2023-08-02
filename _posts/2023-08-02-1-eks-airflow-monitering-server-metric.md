---
title:   "[EKS + Airflow] - monitering server metric"
excerpt: "[EKS + Airflow] - monitering server metric"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - eks
  - airflow
  - HELM
last_modified_at: 2023-07-01T12:06:00+09:00
---

# Background

지금까지 구성한 eks를  운영하기 위해서는 node 및 container들의 resource를 수집하고 관찰하여야 되는데요,
이번시간에는 간단히 k8s의 데이터 수집 및 가시화 오픈소스툴인 prometheus와 grafana를 설치하고 지표들을 탐색해봅시다.

# Contents

## Installation

### prometheus

---

`prometheus`는 `exporter`가 데이터를 수집하고 `prometheus server`가 `exporter`로부터 
정보를 수집합니다.
`prometheus`의 수집 컨테이너가 종료되더라도 수집되던 메트릭의 보존을 위해 storageclass를 사용해줍니다.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update

helm upgrade -i prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="ebs-sc" \
		--set server.persistentVolume.storageClass="ebs-sc"
```

### grafana

---

`grafana`는 오픈소스 메트릭 데이터 시각화 도구입니다. 외부 데이터 소스(prometheus)를 지정할수 있고 쿼리를 통해서 데이터를 동적으로 가지고 옵니다.

`grafana`를 외부에서 접속할수 있도록 service type을 `loadbalancer`로 지정해줍니다.

```bash
helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="ebs-sc"
		--set service.type="LoadBalancer"
```

## Metric

### node resource

---

**`node_memory_MemAvailable_bytes`**

![node-resource](..post_images/eks-airflow-monitering-server-metric/node-resource.jpg)

horizontal pod scaling 설정후 각 정시마다 worker pod가 증가함에 따라 node의 memory가 줄어드는것을 확인할수 있습니다.

### container resource

---

**`container_memory_usage_bytes`**

![node-resource](..post_images/eks-airflow-monitering-server-metric/node-resource.jpg)

각 worker container별  memory 사용량입니다.
task 실행시 limit resource 였던 600Mi가 초과되자 pod가 evicted 되고 다시 생성된것을 볼수 있다.

memory를 600Mi ⇒ 800Mi 로 늘려봅시다.

pod 이름을 my-release-worker를 포함한것으로 filtering 하고 싶을 경우 label filtering에 
`pod =~ ".*my-release-worker.*"` 로 하면됩니다.

**PromQL**

```sql
container_memory_usage_bytes{pod=~".*my-release-worker.*"}
```

### airflow running tasks

---

**`airflow_pool_running_slots_default_pool`**

![node-resource](..post_images/eks-airflow-monitering-server-metric/node-resource.jpg)

각 시간별 airflow에서 실행중인 slots의 갯수입니다.

**PromQL**

```sql
airflow_pool_running_slots_default_pool
```

### airflow worker deployment - replicaset count

---

**`kube_deployment_status_replicas_ready`**

![node-resource](..post_images/eks-airflow-monitering-server-metric/node-resource.jpg)

celery worker들의 replicaset number를 측정한것입니다. 매 정시마다 pod가 scaling out 된것을 확인하실수 있습니다.

**PromQL**

```sql
kube_deployment_status_replicas_ready{deployment=~".*my-release-worker.*"}
```