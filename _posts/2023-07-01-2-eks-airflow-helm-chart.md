---
title:   "[EKS + Airflow] - Airflow 설치하기 (w. Helm chart)"
excerpt: "[EKS + Airflow] - Airflow 설치하기 (w. Helm chart)"
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

지난시간까지 autoscaling 설정 및 efs provisioner가 지정된 storageclass를 구성하였습니다. 지금까지 생성한 scaling, volumn 리소스들을 기반으로 
이번 시간에는 eks cluster에 helm chart 을 이용해서 airflow를 설치해보겠습니다.

aws ecosystem위헤서 구축했기에 관리 point 증가보다는 task의 빠른 실행이 더 우선순위가 되어 celeryexecutor로 설정하였고, scaling option으로 keda 및 cluster autoscaler를 선택하였습니다. 그리고 볼륨 resource은 efs storage class를 사용하여 동적으로 provisioning 하였습니다.

# Contents

## Preparation

airflow helm chart에서 사용할 secret resource들을 생성해줍시다. 우선 airflow에서 metadata storage로 사용할 db를 aws rds로 생성할것이고, 이후 dag를 git sync sidecar 패턴으로 관리하기 위해 secret value들을 생성할것입니다.

### External database for airflow metastore

---

airflow의 metastore를 node 내부에 container로 띄우면 가용성 및 확장성에 제한이 생기므로, aws rds로 metastore 구성하는것을 추천합니다. 

network overhead를 줄이기위해 cluster vpc의 private subnet group 을 생성하고 내부에 rds instance 을 provisioning 합니다.

이후 airflow가 사용할 database를 rds에 생성하고 airflow 전용 계정을 생성해 접근권한을 추가합니다.

```sql
CREATE DATABASE airflow_db;
CREATE USER your_id WITH PASSWORD 'your_pw';
GRANT ALL PRIVILEGES ON DATABASE airflow_db TO your_id;
```

이후 db의 host, 계정 정보를 이용해 helm chart에서 사용할 secret resource를 생성해줍니다.
`CeleryExecutor`를 쓴다면 `result-backend-mydatabase` 에는 `db+` 값을 prefix로 추가해줍니다.

```bash
kubectl create secret -n airflow generic prod_database --from-literal=connection=postgresql://{your_id}:{your_pw}@{host_url}:{port}/airflow_db

kubectl create secret -n airflow generic result-backend-mydatabase --from-literal=connection=db+postgresql://{your_id}:{your_pw}@{host_url}:{port}/airflow_db
```

### Git sync

---

git sync sidecar pattern을 이용해서 dag를 관리하기위해 secret resource를 준비합시다.

ssh key를 생성하고, agent에 key 등록후 git에 ssh key 등록을합니다.

```bash
ssk-keygen -t rsa -b 4096 -C "git sync prep"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
# github: settings > ssh_and_gpg_keys > new_ssh_key > copy and paste ~/.ssh/id_rsa.pub
ssh -T git@github.com
```

이후 gitSshKey와 git-knownhosts 값을 저장하고 이후 helm chart에 이 값들을 설정해주면 됩니다.

```bash
# git-sshkey
base64 ~/.ssh/id_rsa -w 0 > ssh_key_temp.txt

# git-knownhosts
ssh-keyscan -t rsa github.com > github_knownhosts
```

## Helm chart configuration

### Helm install

---

helm 이란 k8s에서 쓰이는 패키니 매니징 tool 입니다. airflow와 같이 여러 component들로 이루어진 서비스를 패키징된 helm chart를 이용하여 쉽게 배포 할수 있습니다. 변경 사항이 있다면 조정된 사항들만 따로 파일로 만들어서 재배포할수도 있습니다.

helm을 설치하고 airflow repo를 설치합니다.

```bash
# install helm
sudo snap install helm --classic

# update airflow repo
helm repo add apache-airflow https://airflow.apache.org
helm repo update
helm search repo airflow
```

### Helm chart 변경사항 작성

---

**executor**

airflow의 표준 helm chart에서 몇가지 수정사항을 정리해봅시다. 파일이름은 `prod_values.yaml` 로 합시다.

첫째로, airflow pod들이 어떻게 실행될지를 결정하는 executor는 celeryExecutor로 정하였습니다. 
주로 celery executor와 kubernetes executor가 많이 쓰이는데 각각의 장단점은 다른 포스팅에서 다루어보도록하겠습니다.

```yaml
executor: "CeleryExecutor"
```

**celery workers**

celery worker의 갯수를 정하고, 리소스들을 명시적으로 설정해줍니다.

```yaml
workers:
  # Number of airflow celery workers in StatefulSet
  replicas: 2
  resources:
    limits:
      cpu: 500m
      memory: 1024Mi
    requests:
      cpu: 500m
      memory: 1024Mi

  # Grace period for tasks to finish after SIGTERM is sent from kubernetes
  terminationGracePeriodSeconds: 600

  logGroomerSidecar:
    resources:
      limits:
        cpu: 100m
        memory: 128Mi
      requests:
        cpu: 100m
        memory: 128Mi
```

auto-scaling 을 위해서 `keda enabled`을 true로 합시다. keda가 airflow celery worker pod의 크기조정을 담당하고 있고 pod의 영속성은 보장할수 없습니다. keda를 사용하기 위해 worker의 persistence를 false로 하고 log storage를 이전에 생성한 efs-sc 스토리지 클래스를 사용하여 생성합니다.

```yaml
workers:  
  # Allow KEDA autoscaling.
  keda:
    enabled: true

    # How often KEDA polls the airflow DB to report new scale requests to the HPA
    pollingInterval: 3

    # How many seconds KEDA will wait before scaling to zero.
    cooldownPeriod: 30

    # Minimum number of workers created by keda
    minReplicaCount: 2

    # Maximum number of workers created by keda
    maxReplicaCount: 10

	persistence:
    # Enable persistent volumes
    enabled: false

logs:
  persistence:
    # Enable persistent volume for storing logs
    enabled: true
    # Volume size for logs
    size: 10Gi
    # If using a custom storageClass, pass name here
    storageClassName: "efs-sc"

config:
  celery:
    worker_concurrency: 2
```

keda 작동시점 :  `template/workers/worker-kedaautoscaler.yaml` 의 `ScaledObject`의 `trigger` 를  보면 airflow db에 접속해서 `running` 또는 `queued` 된 task의 갯수를 `celery.worker_concurrency`로 나누어서 worker가 다 실행할수 있도록 있도록 worker 개수를 조절하고 있습니다.

```yaml
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

참고로  `workers.persistence.enabled`가 true일때는 worker별로 log를 persistent하게 저장하기 위해 celery worker 오브젝트가 `statefulset`임을 확인할수 있습니다. 

```yaml
# template/workers/worker-deployment.yaml 참고
kind: { if $persistence } StatefulSet { else } Deployment { end }
```

**postgresql**

k8s 외부에 metastore를 이용하기 위해 postgresql은 false로 설정해줍니다.
pgbouncer는 true로 설정해주고 자원을 명시적으로 설정해줍니다.

```yaml
postgresql:
  enabled: false
pgbouncer:
  # Enable PgBouncer
  enabled: true
  resources:
    limits:
      cpu: 100m
      memory: 128Mi
    requests:
      cpu: 100m
      memory: 128Mi
```

aws rds로 metadata를 설정하기 위해 위에서 생성한 secret 값들을 data scetion의 key 값들로 지정해줍니다.

```yaml
data:
  metadataSecretName      : prod-mydatabase
  resultBackendSecretName : result-backend-mydatabase
```

**dags**

dag를 git으로 관리하기 위해 gitsync.enabled를 true로 만들어 줍니다. repo, branch와 subpath를 설정해주고 위에서 설정한 `knownhosts`와 `ssh-secret` 값들을 지정해줍니다.

```yaml
dags:
  gitSync:
    enabled: true
    repo: {git_repo}
    branch: {branch_name}
    rev: HEAD
    depth: 1
    subPath: "dags"
    sshKeySecret: ssh-value
    knownHosts: |
      github.com ssh-rsa {ssh-rsa-value}

extraSecrets:
  ssh-value:
    data: |
      gitSshKey: {ssh_key}
```

**webserver**

webserver의 service type을 loadbalancer로 설정하고 유저 계정을 생성합니다.

`triggerer`, `scheduler`, `redis`, `statsd`, `migrateDatabasejob`, `createUserjob` 등등

다른 component들도 resource를 명시적으로 적어줍니다.


## Install airflow via helm charts

### install & update

---

airflow를 설치할 namespace를 생성해줍니다.
```bash
kubectl create namespace airflow
```

위에서 생성한 `prod_airflow_values.yaml` 파일을 통해 `airflow` 를 설치해 줍시다
```bash
helm install my-release apache-airflow/airflow \
  --namespace airflow \
  -f ~/airflow_helm_chart/prod_airflow_values.yaml
```

이후 변동 사항이 있다면 `prod_airflow_values.yaml` 파일을 수정후 `helm upgrade` 를 통해 재배포하면 됩니다.
```bash
helm upgrade my-release apache-airflow/airflow \
  --namespace airflow \
  -f ~/airflow_helm_chart/prod_airflow_values.yaml
```