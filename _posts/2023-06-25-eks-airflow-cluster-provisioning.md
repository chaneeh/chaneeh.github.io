---
title:   "[eks + airflow] - cluster-provisioning"
excerpt: "[eks + airflow] - cluster-provisioning"
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

클라우드상에서 airflow를 이용하기 위해서는 다음과 같은 몇가지 방법들이 있고, 각각 장단점이 있습니다.

1) 단일 ec2

- 설치 및 실행 쉬움
- SPOF 문제 존재
- scalability 제한

2) MWAA

- 관리 포인트에 대한 시간 절약
- 높은 비용 + 관리 포인트 이외의 customizing은 힘듬

3) **eks airflow(w. Helm charts)**

- 가격 최적화 및 성능 customizing 가능
(ec2 ami, airflow version etc…)
- 초기 학습 및 구축 비용

실제 production 환경에서는 가격 대비 효율성 및 customizing 해야하는 부분이 많기 때문에
eks를 이용해서 k8s cluster를 생성하고 airflow를 설치해보는것을 연습해 보겠습니다.

# Contents

## eks cluster provisioning

### 0) Terminology

---

**eks** 란 kubernetes 컨트롤 플레인 또는 노드를 설치, 운영 및 유지 관리할 필요 없이 aws에서 kubernetes를 사용할수 있는 관리형 서비스 입니다. 
**kubernetes**란 컨테이너화된 어플리케이션의 배포, 확장, 관리를 자동화하기 위한 오픈 소스 시스템입니다.

### 1) ec2 provisioning

---

**provisioning via aws cli**

```bash
K8s-mng-system_security_group=$(aws ec2 create-security-group --group-name k8s-mng-system --description "k8s management" --output text)

aws ec2 authorize-security-group-ingress \
    --group-id ${K8s-mng-system_security_group} \
    --protocol tcp \
    --port 22 \
    --cidr {current_cidr}

aws ec2 run-instances \
    --image-id ami-xxxxxxxx  \
    --instance-type t2.large \
    --count 1 \
    --security-group-ids ${K8s-mng-system_security_group} \
    --key-name your-pem \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=K8s-mng-system}]'

```

### 2) cli installation

---

**aws cli install**

```bash
sudo apt-get install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install # You can now run: /usr/local/bin/aws --version
aws --version # aws-aws-cli/2.11.23 Python/3.11.3 Linux/5.15.0-1036-aws exe/x86_64.ubuntu.20 prompt/offcli/2.7.11 Python/3.9.11 Linux/5.13.0-1029-aws exe/x86_64.ubuntu.20 prompt/off
```

**eksctl install**

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version # 0.143.0
```

**kubectl install**

```bash
# Download the kubectl binary for your cluster's Kubernetes version from Amazon S3 using the command for your device's hardware platform
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.25.7/2023-03-17/bin/linux/amd64/kubectl

# Apply execute permissions to the binary.
chmod +x ./kubectl

# Copy the binary to a folder in your PATH. If you have already installed a version of kubectl, then we recommend creating a $HOME/bin/kubectl and ensuring that $HOME/bin comes first in your $PATH.
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH

# (Optional) Add the $HOME/bin path to your shell initialization file so that it is configured when you open a shell
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc

# After you install kubectl, you can verify its version
kubectl version --short --client # Client Version: v1.25.7-eks-0a21954
```

**cli completion option**

```bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### 3) iam configure

---

**user creation**

- Name : eks-mng-user
- access type : programming type
- 기존 정책 연결 : Administratoraccess 권한 부여
- access-key, secret-key store

**aws configure**

```bash
$ aws configure 
# input iam info
# region : ap-northeast-2

# check configuration info
$ aws sts get-caller-identity
```

### 4) eks provisioning & setting

---

**installing eks via eksctl**

```bash
prod_cluster_name="k8s-cluster"

eksctl create cluster \
    --name ${prod_cluster_name} \
    --region ap-northeast-2 \
    --with-oidc \
    --ssh-access \
    --ssh-public-key ${your-key-pair} \
    --managed \
    --spot \
    --instance-types t2.medium,t3.medium,t3.xlarge \
    --nodes 2 \
    --nodes-min 2 \
    --nodes-max 5 \
    --node-volume-size=10
```

eksctl은 이 cli를 통해 cluster가 사용할 vpc, 보안그룹, cluster iam등을 생성합니다.

그 이후 launch-template을 통해서 노드 그룹을 생성합니다.

## reference

[https://github.com/237summit/Kubernetes-on-AWS/blob/main/LAB_k8s_install](https://github.com/237summit/Kubernetes-on-AWS/blob/main/LAB_k8s_install)