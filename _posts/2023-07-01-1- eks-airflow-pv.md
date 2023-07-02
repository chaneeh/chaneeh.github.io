---
title:   "[EKS + Airflow] - PersistentVolume"
excerpt: "[EKS + Airflow] - PersistentVolume"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - eks
  - airflow
  - EFS
last_modified_at: 2023-07-01T12:06:00+09:00
---
# Background

eks cluster에 airflow celery worker pod들이 작업을 수행하면서 asg및 hpa(keda) scaling을 통해 자주 binding 되고 delete될것입니다. 또한 이후 task가 증가함에 따라 worker의 갯수도 유동적으로 변할텐데요, 이러한 상황에서 worker pod의 log를 수집하기 위해 다중 읽기 쓰기가 지원되고 동적으로 provisioning이 되며 multi az에서 파일 시스템이 지원되는 persistentvolumn이 필요합니다.

`aws-ebs` 를 사용하게 된다면 multi AZ에 위치한 airflow worker pod들이 스토리지에 접근할수 없기 때문에 `aws-efs`  프로비저너를 사용한 storage-class를 이용해 동적으로 pv를 구성해보겠습니다.

# Contents

### 0) Terminology

---

**persistentvolumn** 이란 k8s pod의 생명주기와 독립적으로 데이터를 저장할수 있는 클러스터 스토리지입니다.

**efs**란 서버리스 파일 스토리지 제공 서비스입니다. 용량 및 성능을 프로비저닝하지 않거나 관리하지 않고도 파일데이터를 공유할수 있습니다. 여러 서버 및 컴퓨팅 인스턴스에 대한 공통 데이터 소스가 될수 있습니다. 

각 가용영역별로 mount target을 만들어서 진입점을 생성합니다.
multi 가용영역에 provisioning된 pod들이 read & write 할수 있도록 efs를 사용합니다.

### 1)  Binding IAM policy and role

---

iam 정책을 생성하고 iam 역할에 할당합니다. amazon efs 드라이버와 파일 시스템이 상호 작용할수 있도록 권한을 설정합니다.

우선 variable들을 지정합니다

```bash
prod_cluster_name="k8s-cluster"
account_id="your account name"
efs_fs_name="efs_fs"

oidc_value=$(aws eks describe-cluster --name ${prod_cluster_name} --query "cluster.identity.oidc.issuer" --output text)
oidc_value=${oidc_value:8}
```

iam 정책 문서를 다운로드하고 정책을 생성합니다. 
신뢰 정책을 생성한다음 k8s 서비스 계정에 AssumeRoleWithWebIdentity 작업을 부여합니다.

```bash

curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/v1.2.0/docs/iam-policy-example.json
mv iam-policy-example.json ~/pv_installation/efs_csi/iam-policy-example.json

aws iam create-policy \
    --policy-name AmazonEKS_EFS_CSI_Driver_Policy \
    --policy-document file://~/pv_installation/efs_csi/iam-policy-example.json

cat <<EOF > ~/trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${account_id}:oidc-provider/${oidc_value}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${oidc_value}:sub": "system:serviceaccount:kube-system:efs-csi-controller-sa"
        }
      }
    }
  ]
}
EOF
```

해당 신뢰 정책을 기반으로 역할을 생성하고 정책을 연결합니다.

```bash
aws iam create-role \
  --role-name AmazonEKS_EFS_CSI_DriverRole \
  --assume-role-policy-document file://~/pv_installation/efs_csi/trust-policy.json

aws iam attach-role-policy \
  --policy-arn arn:aws:iam::${account_id}:policy/AmazonEKS_EFS_CSI_Driver_Policy \
  --role-name AmazonEKS_EFS_CSI_DriverRole
```

### 2) Creating AWS EFS driver

---

메니페스트를 다운로드하여 퍼블릭 ecr registry에 저장된 이미지를 사용하여 efs 드라이버를 설치합니다.

```bash
kubectl kustomize "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.3" > ~/pv_installation/efs_csi/public-ecr-driver.yaml
```

public-ecr-driver의 efs-csi-controller-sa 서비스 어카운트에 위에서 생성한 iam 역할의 arn 주석을 답니다.

주석을 다는이유는 eks에서 지원하는 `IAM Roels for Service Accounts`의 동작원리때문인데요, k8s에서 service account를 설정할때 role arn을 주석으로 달아주고(`eks.amazon.com/role-arn`), 이후 eks 내부에서 `admission controller` 이 sa의 role arn annotation을 기반으로 aws 권한을 부여하기 때문입니다.

```bash
sed -i -z 's/app.kubernetes.io\/name: aws-efs-csi-driver\n  name: efs-csi-controller-sa/app.kubernetes.io\/name: aws-efs-csi-driver \
  annotations: \
    eks.amazonaws.com\/role-arn: arn:aws:iam::'"${account_id}"'':role\/AmazonEKS_EFS_CSI_DriverRole \
  name: efs-csi-controller-sa/' ~/pv_installation/efs_csi/public-ecr-driver.yaml
```

aws efs csi driver를 배포하기 위해 변경된 매니페스트를 적용하여 줍니다.

```bash
kubectl apply -f ~/pv_installation/efs_csi/public-ecr-driver.yaml
```

### 3) EFS File System mount target

---

amazon eks cluster의 vpc id와 cidr block 을 가져옵니다.

```bash
vpc_id=$(aws eks describe-cluster --name ${prod_cluster_name} --query "cluster.resourcesVpcConfig.vpcId" --output text)

cidr_block=$(aws ec2 describe-vpcs --vpc-ids ${vpc_id} --query "Vpcs[].CidrBlock" --output text)
```

보안그룹을 생성하고 해당 vpc의 cidr block에 대해서 인바운드 NFS 트래픽을 허용하는 인바운드 규칙을 생성합니다.

```bash
sg_id=$(aws ec2 create-security-group --description efs-test-sg --group-name efs-sg --vpc-id ${vpc_id} --output text)

aws ec2 authorize-security-group-ingress --group-id ${sg_id} --protocol tcp --port 2049 --cidr ${cidr_block}
```

파일 시스템을 만들고 k8s 클러스터 노드가 있는 각 서브넷에 대한 
`mount target`을 추가합니다. 위에서 생성한 NFS 트래픽 허용 보안그룹을 `mount target`에 적용합니다.

```bash
aws efs create-file-system --creation-token eks-efs --tags Key=Name,Value=${efs_fs_name}

fs_id=$(aws efs describe-file-systems --query "FileSystems[?Name=='${efs_fs_name}'].FileSystemId" --output text)

vpc_id_filter=$(echo "Name=vpc-id,Values=${vpc_id}")

subnet_ids=$(aws ec2 describe-subnets --filters ${vpc_id_filter} 'Name=tag:aws:cloudformation:logical-id,Values=SubnetPublicAPNORTHEAST2A, SubnetPublicAPNORTHEAST2B, SubnetPublicAPNORTHEAST2C, SubnetPublicAPNORTHEAST2D' --query "Subnets[*].SubnetId" --output text)
#	[subnet-{id_1}       subnet-{id_2}       subnet-{id_2}]

for subnet_id in ${subnet_ids}
do
    aws efs create-mount-target --file-system-id ${fs_id} --subnet-id ${subnet_id} --security-group ${sg_id}
done
```

### 4) Storage class Manifest file

---

위에서 생성한 efs file-system id를 이용하여 airflow에서 사용할 storage class manifest file을 생성하고 k8s 리소스를 생성해 줍니다.

```bash
fs_id=$(aws efs describe-file-systems --query "FileSystems[?Name=='${efs_fs_name}'].FileSystemId" --output text)

cat <<EOF > ~/customize_yaml/efs-sc.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: ${fs_id}
  directoryPerms: "700"
  gidRangeStart: "1000" # optional
  gidRangeEnd: "2000" # optional
  basePath: "/prod_airflow" # optional
EOF

kubectl create -f ~/customize_yaml/efs-sc.yaml
```

이후 airflow helm chart에서 pvc를 생성하면 `efs-sc` 라는 storage-class를 이용해 pv를 동적으로 생성할 것입니다.

## Reference

---

**EFS**

[Amazon EFS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)

[How do I use persistent storage in Amazon EKS?](https://repost.aws/ko/knowledge-center/eks-persistent-storage)