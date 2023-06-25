---
title:   "[eks + airflow] - auto-scaling"
excerpt: "[eks + airflow] - auto-scaling"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - Blog
  - eks
  - airflow
  - TOC
last_modified_at: 2023-06-25T12:06:00+09:00
---
# Background

eks에 airflow를 리소스 효율적으로 관리하기 위해서는 크게 pod scaling 및 node scaling이 필요합니다. 

이번 포스트는 pod와 node를 scaling 하기 위해서 keda와 k8s cluster autoscaler를 사용해보겠습니다.

# Contents

## EKS scaling

### 1) cluster autoscaling(ca)

---

eks에 autoscaling 하기 위해 몇가지 option들이 있습니다.

cluster autoscaler(ca)은 unscheduled 된 pod가 있을때 ca가 aws asg을 통해 scale out을 합니다. 그 특성으로 인해 몇가지 단점이 있습니다, aws asg를 통하기 때문에 scale out시 속도가 느리고 노드 그룹 변동/확장이 있을때 노드 그룹 생성과 ca를 각각 관리해야합니다.

이러한 문제들을 해결하기 위해 karpenter라는 groupless auto scaling 오픈 소스가 나왔습니다. karpenter는 provisioner 자체가 노드 그룹의 역할을 하기때문에 노드 그룹 변동시 관리 포인트가 줄고, aws asg를 통하지 않기때문에 scale out시 속도도 더 빠릅니다.

자세한 비교 (cluster autoscaler vs karpenter)는 다른 포스트에서 자세히 다루어보도록 하겠습니다.

이번 튜토리얼에서는 airflow는 실제 연산이 아니라 job scheduling의 역할을 하고 있기 때문에 node-group 및 노드의 다양성이 크게 중요하지는 않았습니다. default node group의 scaling으로도 scheduling을 잘 해낼수 있었기에 cluster autoscaling을 선택하였습니다.

**cluster autoscaling install**

script 내 variable들 설정합니다.

```bash
prod_cluster_name="dev-k8s-demo-v3"
account_id="343647978193"
```

asg를 설정변경할수 있는 policy를 생성하고 해당 policy 권한이 부여된 k8s service account를 생성합니다.

eks에서 k8s service account와 iam role을 mapping하기 위해 `IAM Roles for Service Accounts (IRSA)`를 지원하고  eksctl을 이용해서 iam role과 service account pair를 쉽게 생성할수 있습니다.

동작원리는 k8s에서 service account를 설정할때 role arn을 주석으로 달아주고, (`eks.amazon.com/role-arn`) 이후 eks 내부에서 `admission controller` 이 annotation 기반으로 권한을 부여합니다.

```bash
cat <<EOF > cluster-autoscaler-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/k8s.io/cluster-autoscaler/${prod_cluster_name}": "owned"
                }
            }
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeAutoScalingGroups",
                "ec2:DescribeLaunchTemplateVersions",
                "autoscaling:DescribeTags",
                "autoscaling:DescribeLaunchConfigurations"
            ],
            "Resource": "*"
        }
    ]
}
EOF

aws iam create-policy \
    --policy-name AmazonEKSClusterAutoscalerPolicy \
    --policy-document file://cluster-autoscaler-policy.json

eksctl create iamserviceaccount \
  --cluster=${prod_cluster_name} \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::${account_id}:policy/AmazonEKSClusterAutoscalerPolicy \
  --override-existing-serviceaccounts \
  --approve  
```

cluster autoscaler yaml을 다운받고 cluster이름을 설정해준후 변경된 메니페스트를 적용합니다.

```bash

curl -o cluster-autoscaler-autodiscover.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

sed -i "s/<YOUR CLUSTER NAME>/${prod_cluster_name}/g" cluster-autoscaler-autodiscover.yaml

kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

cluster-autoscaler deployment에 safe_to_evict 주석을 추가합니다. 

```bash

kubectl patch deployment cluster-autoscaler \
  -n kube-system \
  -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'

```

cluster name 및 container의 command에 몇가지 수정사항을 반영해서 다시 patch를 진행합니다.
이후 image 버전도 최신버전으로 바꾸어 줍니다.

```bash

cat <<EOF > cluster-autoscaler-deployment-update.yaml
spec:
  template:
    spec:
      containers:
        - name: cluster-autoscaler
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/${prod_cluster_name}
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
EOF

kubectl patch deployment cluster-autoscaler \
  -n kube-system \
  --patch-file cluster-autoscaler-deployment-update.yaml

kubectl set image deployment cluster-autoscaler \
  -n kube-system \
  cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v1.23.1    

```

### 2) KEDA(event-driven hpa)

---

`KEDA`란 k8s based event driven autoscaler 입니다. 

Horizontal Pod Autoscaler와 같이 동작하고 hpa를 overwrite하거나 복제없이 extend할수 있습니다. keda를 통해 사용자가 원하는 event-driven scale을 사용할수 있습니다.

다양한 scaler 및 trigger plugin을 제공하고 ScaledObject crd를 통해 trigger, scaler에 관한 object를 명시합니다.

**installing helm & updating keda repo**

```bash
# installing helm
sudo snap install helm --classic

# update keda repo
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
```

**installing horizontal autoscaling**

```bash
# create keda namespace
kubectl create namespace keda

# installing keda via helm
helm install keda kedacore/keda \
    --namespace keda \
    --version "v2.0.0"
```

**Enabling keda in airflow helm chart**

```bash
executor: "CeleryExecutor"
# Airflow Worker Config
workers:
  # Allow KEDA autoscaling.
  keda:
    enabled: true
    # How often KEDA polls the airflow DB to report new scale requests to the HPA
    pollingInterval: 3
    # How many seconds KEDA will wait before scaling to zero.
    # Note that HPA has a separate cooldown period for scale-downs
    cooldownPeriod: 30
    # Min/Max number of workers created by keda
    minReplicaCount: 4
    maxReplicaCount: 60
```

**keda trigger**

`ScaledObject`의 `trigger` 를  보면 airflow db에 접속해서 `running` 또는 `queued` 된 task의 갯수를 `celery.worker_concurrency`로 나누어서 worker가 다 실행할수 있도록 worker 개수를 조절하고 있습니다.

```sql
SELECT ceil(COUNT(*)::decimal / {{ .Values.config.celery.worker_concurrency }})
FROM task_instance
WHERE (state='running' OR state='queued')
{- if eq .Values.executor "CeleryKubernetesExecutor" }
AND queue != '{ .Values.config.celery_kubernetes_executor.kubernetes_queue }'
{- end }
```

## Reference

**auto-scaler, karpenter**

[AWS EKS 의 Cluster Autoscaler 설정](https://velog.io/@airoasis/eks-cluster-autoscaler)

[Cluster Autoscaler on AWS](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)

[eksctl - IAM Roles for Service Accounts](https://eksctl.io/usage/iamserviceaccounts/)

[EKS Karpenter를 활용한 Groupless AutoScaling](https://swalloow.github.io/eks-karpenter-groupless-autoscaling/)

**keda**

[service bus 기반 hpa 구성](https://velog.io/@pwcasdf/Azure-Kubernetes-Service-KEDA-service-bus-%EA%B8%B0%EB%B0%98-hpa-%EA%B5%AC%EC%84%B1)

[keda 홈페이지](https://keda.sh/)