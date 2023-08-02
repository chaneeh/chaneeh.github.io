---
title:   "[EKS + Airflow] - CA vs KARPENTER"
excerpt: "[EKS + Airflow] - CA vs KARPENTER"
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

k8s 환경위에서 airflow를 운영하다보면 task가 증가함에 따라 컴퓨팅 용량이 확장되어야할 필요성이 생깁니다. celery worker concurrency를 늘린다던가 resource intensive한 task가 증가하는 경우가 그렇습니다.
resource 할당량이 커진 worker pod를 scheduling 하기 위해 node의 type, resource size도 바뀌어야합니다. 

eks 클러스터 내에서 노드 scaling을 위한 도구로써 autoscaler와 karpenter가 있는데요, 이번시간에는 CA(Kubernetes Cluster Autoscaler)와 Karpenter를 사용하여 pod의 resource가 증가하면서 확장된 용량의 node를 scaling하는 예시를 관찰해보겠습니다. 실험을 통해 어떤 도구가 안전성 및 관리포인트가 적은지 알아보겠습니다.

ca와 karpenter에 대해서 간단하게 정리를 해보자면 다음과 같습니다.

ca는 스케쥴링 되지 못한 파드가 있으면 auto scaling group(이하 asg)에 속한 노드를 클러스터에 추가함으로써 pod가 사용할 리소스를 확보합니다. 한가지 주의할점은 ca는 asg에 속한 노드들의 타입은 모두 수동으로 설정해야 하거나 같은 vcpu와 메모리를 가진다고 가정합니다. 그렇기에 스팟 인스턴스 용량을 안정적으로 확보하거나 변화하는 용량에 대비할려면 여러 사이즈와 패밀리 노드 asg를 조합해야 합니다. 또한 여러 asg들의 우선순위를 expander를 통해 `random`, `least-waste`, `price`등 여러 옵션을 수동을 조정할수 있습니다.

karpenter는 ca와 마찬가지로 스케쥴링 되지 못한 pod가 있을때 노드를 증설합니다. 다만 ca와 다르게 노드그룹을 기반으로 노드를 증설하는게 아닌 ec2에 직접 api call을 함으로써 증설을 합니다. 이후 existing capacity들의 resource들을 비교하여 현재 운영되는 pod들에 적합한 optimized capacity도 karpenter가 관리하고 변경합니다. 또한 클러스터 용량 조절을 위해 provisioner라는 crd를 사용하고 allocation-stratagy는 용량 타입(spot, on-demand)에 따라 달라집니다.
spot instance capacity type을 사용하면 price-capacity-optimized allocation strategy를 사용하여 node-interruption이 적은 node type중에 lowest price node를 선택  합니다.
on-demand capacity type일때는 기본적으로 capacity가 보장이 되기때문에 가격이 제일 적게드는 lowest price allocation strategy를 사용합니다.

![1.png](https://raw.githubusercontent.com/chaneeh/chaneeh.github.io/master/img/eks-airflow-ca-karpenter/1.png)

# Contents

## Experiment

실험을 위해 CA와 karpenter가 `celery worker pod resource`가 증가하였을때 확장된 컴퓨팅 용량을 scaling 하는 과정을 비교해보겠습니다. 

비교 기준은 다음과 같습니다.
- pod resource가 증가해도 k8s 관리/변경 포인트를 적게 유지하면서 다음 크기 용량의 노드로 증설시켜주는가?

### 비교 환경 세팅

---

기존 celery worker는 2vcpu 4G memory node에 할당될수 있는 resource를 request하지만, 
재조정된 celery worker 는 2vcpu가 초과된 resource request를 설정함으로써 확장된 컴퓨팅 용량에만 scheduling 되게 설정하였습니다. 실험을 위해 instance family는 t group으로 고정하였습니다.

**기존 `celery worker resource`**

```yaml
workers:
  resources:
    limits:
      cpu: 1200m
      memory: 1400Mi
    requests:
      cpu: 1200m
      memory: 1400Mi
```

**재조정된 `celery worker resource`**

```yaml
workers:
  resources:
    limits:
      cpu: 2400m
      memory: 2800Mi
    requests:
      cpu: 2400m
      memory: 2800Mi
```

### Cluster Autoscaler

---

**CA 초기 세팅**

CA의 asg들간의 우선순위는 `least-waste`로 설정하였습니다.

`cluster_autoscaler expander - least-waste`

```yaml
...
spec:
  template:
    spec:
      containers:
        - name: cluster-autoscaler
          command:
            - ./cluster-autoscaler
						...
            - --expander=least-waste
```

airflow의 기본 pod들을 위해 `ng-basic` 이라는 on-demand asg을 기본으로 세팅해 놓았습니다.

```yaml
aws autoscaling describe-auto-scaling-groups \
    --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='${CLUSTER_NAME}']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" \
    --output table
---------------------------------------------------------------------------
|                        DescribeAutoScalingGroups                        |
+--------------------------------------------------------+----+-----+-----+
|  eks-ng-basic-0ec48b61-1b84-04d9-4303-0d94f005eb4f     |  3 |   3 |   3 |
+--------------------------------------------------------+----+-----+-----+
```

**[Before worker scale up]**

2vcpu 4G memory spec의 t3a.medium type으로 구성된 asg을 worker pod의 scheduling node로 사용하였습니다. 10개의 dag가 trigger 되고 5분후, 10개의 dag를 실행시키기 위한 10개의 worker pod가 실행되었고, worker pod당 1.2vcpu를 request 하기 때문에 한 노드당 최대 한개의 worker pod가 실행됩니다.  `eks-ng-spot-2vcpu` node-group의 desired capacity가 10개가 된것을 확인할수 있습니다.

```bash
eksctl create nodegroup \
--cluster="k8s-cluster" --region="ap-northeast-2" \
--managed --spot --name="ng-spot-2vcpu" \
--instance-types=t3a.medium \
--nodes 1 --nodes-min 1 --nodes-max 10
```

```bash
aws autoscaling describe-auto-scaling-groups \
    --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='${CLUSTER_NAME}']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" \
    --output table

-----------------------------------------------------------------------------
|                         DescribeAutoScalingGroups                         |
+----------------------------------------------------------+----+-----+-----+
|  eks-ng-479fccee-0ec48b61-1b84-04d9-4303-0d94f005eb4f    |  3 |  3  |  3  |
|  eks-ng-spot-2vcpu-36c4bbb1-d9d4-c550-c098-4ec5a41d6024  |  1 |  10 |  10 |
+----------------------------------------------------------+----+-----+-----+
```

**[After worker scale up] without t3a.xlarge ⇒ pending**

기존의 request에서 2배를 늘려 2vcpu spec을 가진 노드의 spec을 초과하게 설정하였습니다.
해당 request pod를 binding시킬려면 t family 중 다음 사이즈인 t3a.xlarge 이상의 노드가 있어야합니다.

**increasing worker pod resource**

```yaml
workers:
  resources:
    limits:
      cpu: 2400m
      memory: 2800Mi
    requests:
      cpu: 2400m
      memory: 2800Mi
```

worker pod는 2.4cpu를 요청하고 있고 ca는 현재 2vcpu 노드로 구성된 ng-spot-2vcpu asg그룹만 있습니다. 더큰 용량의 asg를 추가하지 않고 dag 10개가 trigger 되면 어떻게 될까요?

hpa에 의해서 worker replicaset은 10개로 설정이 되었지만 resource 부족으로 node에 scheduled 되지 않았고 worker의 state가 pending 임을 확인할수 있습니다.  cpu credit 소모(일시적으로 추가 cpu 할당)를 통해 t3a.medium instance에서 몇개의 pod가 running하고 있는것을 확인할수 있습니다. 다만 대부분의 dag들은 timeout으로 fail이 났습니다.

```yaml
**kubectl get pods -n airflow**

NAME                                    READY   STATUS        RESTARTS        AGE
my-release-scheduler-786fc5779d-zsg2j   3/3     Running       2 (2d16h ago)   6d5h
my-release-worker-66d5cccdc4-2jpc7      0/2     Pending       0               10m
my-release-worker-66d5cccdc4-hdv6s      0/2     Pending       0               23m
...
my-release-worker-66d5cccdc4-sgw7f      0/2     Pending       0               9m53s
my-release-worker-89b8f5bb6-r5ln8       2/2     Running       0               82m
```

```yaml
----------------------------------------------------------------------------
|                         DescribeAutoScalingGroups                        |
+----------------------------------------------------------+----+-----+----+
|  eks-ng-479fccee-0ec48b61-1b84-04d9-4303-0d94f005eb4f    |  3 |  3  |  3 |
|  eks-ng-spot-2vcpu-36c4bbb1-d9d4-c550-c098-4ec5a41d6024  |  1 |  10 |  4 |
+----------------------------------------------------------+----+-----+----+
```

**[After worker scale up] with t3a.xlarge asg⇒ expanding to t3a.xlarge group!**

위의 예시를 통해, ca에서 용량 확장이 필요할 경우 해당 용량을 가진 asg를 명시적으로 추가해주어야함을 알수 있습니다. 이제 4vcpu spec t3a.xlarge로 구성된 ng-spot-4vcpu asg을 추가해봅시다.

```bash
eksctl create nodegroup \
--cluster="k8s-cluster" --region="ap-northeast-2" \
--managed --spot --name="ng-spot-4vcpu" \
--instance-types=t3a.xlarge \
--nodes 1 --nodes-min 1 --nodes-max 10
```

이제 scale out이 가능한 asg에 ng-spot-4vcpu asg가 추가되었습니다.

```yaml
----------------------------------------------------------------------------
|                         DescribeAutoScalingGroups                        |
+----------------------------------------------------------+----+-----+----+
|  eks-ng-479fccee-0ec48b61-1b84-04d9-4303-0d94f005eb4f    |  3 |  3  |  3 |
|  eks-ng-spot-2vcpu-36c4bbb1-d9d4-c550-c098-4ec5a41d6024  |  1 |  10 |  1 |
|  eks-ng-spot-4vcpu-c8c4bbc9-787c-b8b2-8235-a04c20cbd403  |  1 |  10 |  1 |
+----------------------------------------------------------+----+-----+----+
```

10개의 dag를 실행시키기 위해 2.4 vcpu를 요청하는 worker pod 10개가 모두 running status로 되었습니다.
또한 4vcpu asg도 안정적으로 10개로 늘어났음을 확인할수 있습니다.

```bash
NAME                                    READY   STATUS    RESTARTS        AGE
my-release-scheduler-786fc5779d-zsg2j   3/3     Running   2 (2d16h ago)   6d5h
my-release-worker-66d5cccdc4-29s2r      2/2     Running   0               2m51s
my-release-worker-66d5cccdc4-2m4c6      2/2     Running   0               2m51s
my-release-worker-66d5cccdc4-6xsxw      2/2     Running   0               2m51s
my-release-worker-66d5cccdc4-hdv6s      2/2     Running   0               39m
my-release-worker-66d5cccdc4-mj4dc      2/2     Running   0               3m6s
my-release-worker-66d5cccdc4-mqvvr      2/2     Running   0               2m36s
my-release-worker-66d5cccdc4-p95k2      2/2     Running   0               3m6s
my-release-worker-66d5cccdc4-qk4ml      2/2     Running   0               2m51s
my-release-worker-66d5cccdc4-slpvx      2/2     Running   0               3m6s
my-release-worker-66d5cccdc4-x497n      2/2     Running   0               2m36s
```

```yaml
-----------------------------------------------------------------------------
|                         DescribeAutoScalingGroups                         |
+----------------------------------------------------------+----+-----+-----+
|  eks-ng-479fccee-0ec48b61-1b84-04d9-4303-0d94f005eb4f    |  3 |  3  |  3  |
|  eks-ng-spot-2vcpu-36c4bbb1-d9d4-c550-c098-4ec5a41d6024  |  1 |  10 |  1  |
|  eks-ng-spot-4vcpu-c8c4bbc9-787c-b8b2-8235-a04c20cbd403  |  1 |  10 |  10 |
+----------------------------------------------------------+----+-----+-----+
```

이번 실험을 통해 확인할수 있듯이 pod request가 늘어남에 따라 더큰 용량의 노드가 필요할 경우 ca는 target node type이 명시된 asg를 추가해주어야함을 확인할수 있었습니다.

이제 karpenter의 경우는 어떻게 노드 용량을 변화시키는지 관찰해봅시다.

### Karpenter - provisionor

---

결론부터 말씀드리자면 karpenter는 초기 provisioner만 설정해놓으면 더큰 용량의 노드도 별도의 관리 포인트 없이 scale out이 가능합니다.

이번 실험에 t instance family중에 2,4,8,16,32 cpu를 가진 spot instance들을 사용하기 위해 다음과 같이 provisionor를 작성해주기만 하면 다양한 용량을 커버하는 scale out 준비는 끝납니다.

**provisionor 1 - only one needed**

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
    - key: "karpenter.k8s.aws/instance-cpu"
      operator: In
      values: ["2", "4", "8", "16", "32"]
    - key: "karpenter.k8s.aws/instance-category"
      operator: In
      values: ["t"]
  limits:
    resources:
      cpu: 1000
  providerRef:
    name: default
  consolidation: 
    enabled: true
```

**[Before worker scale up]** 

worker를 scale up하기 전 10개의 dag를 위해  karpenter가 10개의 t3a.medium을 증설한것을 확인할수 있습니다.

```yaml
watch kubectl get nodes -L karpenter.sh/capacity-type -L karpenter.k8s.aws/instance-family -L karpenter.k8s.aws/instance-size

Every 2.0s: kubectl get nodes -L karpenter.sh/capacity-type -L karpenter.k8s.aws/instance-family -L karpenter.k8s.aws/instance-size           

ip-192-168-15-23.ap-northeast-2.compute.internal    Ready    <none>   37s     v1.25.11-eks-a5565ad   spot            t3a               medium
ip-192-168-27-208.ap-northeast-2.compute.internal   Ready    <none>   53s     v1.25.11-eks-a5565ad   spot            t3a               medium
ip-192-168-30-158.ap-northeast-2.compute.internal   Ready    <none>   20s     v1.25.11-eks-a5565ad   spot            t3a               medium
ip-192-168-64-37.ap-northeast-2.compute.internal    Ready    <none>   52s     v1.25.11-eks-a5565ad   spot            t3a               medium
ip-192-168-66-121.ap-northeast-2.compute.internal   Ready    <none>   20s     v1.25.11-eks-a5565ad   spot            t3a               medium
ip-192-168-66-174.ap-northeast-2.compute.internal   Ready    <none>   36s     v1.25.11-eks-a5565ad   spot            t3a               medium
ip-192-168-73-175.ap-northeast-2.compute.internal   Ready    <none>   3m12s   v1.25.11-eks-a5565ad   spot            t3a               medium
ip-192-168-79-63.ap-northeast-2.compute.internal    Ready    <none>   33s     v1.25.11-eks-a5565ad   spot            t3a               medium
ip-192-168-83-161.ap-northeast-2.compute.internal   Ready    <none>   52s     v1.25.11-eks-a5565ad   spot            t3a               medium
ip-192-168-93-120.ap-northeast-2.compute.internal   Ready    <none>   32s     v1.25.11-eks-a5565ad   spot            t3a               medium
```

**[After worker scale up]**
2.4vcpu를 요청하는 pod를 schedule하기 위해 4vcpu 이상의 node size가 필요합니다.
karpenter는 spot instance 경우 price-capacity-optimized allocation strategy를 따르기에 제일 node-interruption이 적고 lowest price인 t3a.xlarge를 자동으로 선정해서 scale out 해줍니다.

```yaml
watch kubectl get nodes -L karpenter.sh/capacity-type -L karpenter.k8s.aws/instance-family -L karpenter.k8s.aws/instance-size

Every 2.0s: kubectl get nodes -L karpenter.sh/capacity-type -L karpenter.k8s.aws/instance-family -L karpenter.k8s.aws/instance-size           

NAME                                                 STATUS     ROLES    AGE    VERSION                CAPACITY-TYPE   INSTANCE-FAMILY   INSTANCE-SIZE
ip-192-168-112-197.ap-northeast-2.compute.internal   Ready      <none>   10s    v1.25.11-eks-a5565ad   spot            t3a               xlarge
ip-192-168-113-169.ap-northeast-2.compute.internal   Ready      <none>   26s    v1.25.11-eks-a5565ad   spot            t3a               xlarge
ip-192-168-115-227.ap-northeast-2.compute.internal   Ready      <none>   38s    v1.25.11-eks-a5565ad   spot            t3a               xlarge
ip-192-168-28-48.ap-northeast-2.compute.internal     Ready      <none>   21s    v1.25.11-eks-a5565ad   spot            t3a               xlarge
ip-192-168-38-250.ap-northeast-2.compute.internal    Ready      <none>   34s    v1.25.11-eks-a5565ad   spot            t3a               xlarge
ip-192-168-57-156.ap-northeast-2.compute.internal    Ready      <none>   23s    v1.25.11-eks-a5565ad   spot            t3a               xlarge
ip-192-168-63-177.ap-northeast-2.compute.internal    Ready.     <none>   9s     v1.25.11-eks-a5565ad   spot            t3a               xlarge
ip-192-168-76-40.ap-northeast-2.compute.internal     Ready      <none>   22s    v1.25.11-eks-a5565ad   spot            t3a               xlarge
ip-192-168-80-95.ap-northeast-2.compute.internal     Ready      <none>   41h    v1.25.11-eks-a5565ad   spot            t3a               xlarge
ip-192-168-86-190.ap-northeast-2.compute.internal    Ready      <none>   37s    v1.25.11-eks-a5565ad   spot            t3a               xlarge
```

# Conclusion

이번 시간에 pod request 증가에 따라 더 큰 용량의 node를 scaling 해야하는 실험을 해보았습니다.  
이번 예시에서 karpenter가 scaling 시 provisioner가 자동으로 scaling 해주기에 관리포인트가 더 적음을 확인할수 있습니다.  다만 ca가 여러 환경의 node scaling을 지원해주고 레퍼런스도 많은 반면 karpenter는 aws만 지원해주고 참고할 문서가 많이 없는 편입니다. 

운영하고자 하는 환경에 맞게 scaling option을 선택하시면 될것 같습니다만, aws에서 pod resource가 자주 변하는 환경에서 운영하고 계시다면 관리포인트를 줄이기 위해 karpenter를 고려해보는것도 좋을것 같습니다.

## Reference

---

**[Amazon EKS 클러스터를 비용 효율적으로 오토스케일링하기](https://aws.amazon.com/ko/blogs/tech/amazon-eks-cluster-auto-scaling-karpenter-bp/)**

[**How does Karpenter dynamically select instance types?**](https://karpenter.sh/preview/faq/)