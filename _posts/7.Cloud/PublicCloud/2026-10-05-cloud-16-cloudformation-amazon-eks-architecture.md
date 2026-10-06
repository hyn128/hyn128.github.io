---
title: CloudFormation과 Amazon EKS 구조
description: CloudFormation의 Stack·Template 동작과 Terraform의 차이를 정리하고 Amazon EKS의 Control Plane, Data Plane, VPC CNI, IAM과 Load Balancer 연동 구조를 확인합니다
date: 2026-10-05
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> CloudFormation은 AWS Resource 구성을 Template으로 정의하고 Stack 단위로 생성·변경·삭제하는 AWS 전용 IaC Service입니다. Amazon EKS는 Kubernetes Control Plane을 AWS가 관리하며, 사용자는 EC2 Managed Node Group이나 Fargate 같은 Data Plane을 선택합니다. 이 글에서는 구축 명령을 실행하기 전에 두 Service의 역할과 EKS Network·인증·외부 노출 구조를 정리합니다.

## 1. CloudFormation의 역할

---

CloudFormation은 AWS Resource의 원하는 상태를 JSON 또는 YAML Template으로 정의합니다. Template을 실행하면 CloudFormation이 Resource를 만들고 Stack 단위로 상태를 추적합니다.

```text
CloudFormation Template
  ↓ Create stack
CloudFormation Stack
  ├── VPC
  ├── Subnet
  ├── Route Table
  └── Internet Gateway
```

| 용어 | 역할 |
| --- | --- |
| Template | 생성할 Resource, Parameter, Mapping, Condition과 Output을 선언합니다. |
| Stack | Template으로 생성한 Resource 묶음과 현재 상태입니다. |
| Change Set | Stack 변경 전에 생성·수정·삭제될 Resource를 검토합니다. |
| Drift | 실제 Resource 설정이 Stack Template과 달라진 상태입니다. |
| Output | 생성된 VPC ID나 Subnet ID처럼 후속 작업에서 사용할 값을 반환합니다. |

Infrastructure를 Template으로 관리하면 같은 구성을 반복해서 만들 수 있고, Git에서 변경 이력을 검토할 수 있습니다. 수동 입력을 줄일 수 있지만 Template 오류, IAM 권한, Service Quota와 Region별 가용성까지 자동으로 해결되는 것은 아닙니다.

## 2. CloudFormation과 Terraform 비교

---

| 구분 | AWS CloudFormation | Terraform |
| --- | --- | --- |
| 제공사 | AWS | HashiCorp |
| 지원 범위 | AWS Resource 중심 | Provider를 통해 Cloud와 외부 Service 지원 |
| 선언 언어 | JSON, YAML | HCL, JSON |
| 상태 관리 | AWS가 Stack 상태를 관리 | 사용자가 State Backend와 Locking을 구성 |
| AWS 신규 기능 반영 | AWS Resource Provider를 통해 제공 | AWS Provider Version에 따라 반영 |
| Module 생태계 | Nested Stack, Module, Registry | Module Registry와 Provider 생태계 |

CloudFormation은 별도의 State File을 직접 운영하지 않습니다. Terraform은 State를 Local 또는 S3 같은 Remote Backend에 저장하므로 접근 통제와 동시 실행 방지 구성이 필요합니다.

## 3. S3 Bucket Template 예제

---

다음 Template은 Bucket 이름을 Parameter로 받고 S3 Bucket을 생성합니다. S3 Bucket 이름은 전 세계에서 고유해야 하므로 고정된 예제 이름을 그대로 사용하지 않습니다.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Create an S3 bucket for an IaC example

Parameters:
  BucketName:
    Type: String
    Description: Globally unique S3 bucket name

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      BucketName: !Ref BucketName
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        IgnorePublicAcls: true
        BlockPublicPolicy: true
        RestrictPublicBuckets: true

Outputs:
  BucketName:
    Value: !Ref MyS3Bucket
```

`DeletionPolicy: Retain`은 Stack을 삭제해도 Bucket을 보존합니다. 실습 Resource까지 함께 삭제하려면 Retain 정책과 Bucket 내부 Object 처리 방식을 먼저 결정합니다.

Template의 기본 문법을 AWS API로 확인합니다.

```bash
aws cloudformation validate-template \
  --template-body file://s3-bucket.yaml
```

검증이 성공해도 현재 IAM 권한, 이름 중복과 Service Quota 때문에 Stack 생성이 실패할 수 있습니다. 생성 전 Change Set과 Stack Event를 함께 확인합니다.

## 4. Amazon EKS의 관리 경계

---

> Amazon EKS는 Kubernetes API Server와 `etcd`를 포함한 Control Plane을 AWS가 운영하는 Managed Kubernetes Service입니다.

AWS는 EKS를 re:Invent 2017에서 발표했고 서울 Region에는 2019년에 제공하기 시작했습니다.

Kubernetes Cluster는 Control Plane과 Data Plane으로 나뉩니다.

```text
Operator
  ↓ kubectl 요청
EKS Control Plane
  ├── kube-apiserver
  ├── etcd
  ├── scheduler
  └── controller manager
        ↓ Pod 배치 결정과 상태 조정
Data Plane
  ├── Worker Node 1 → kubelet → Pod
  └── Worker Node 2 → kubelet → Pod
```

Control Plane은 API 요청을 인증·인가하고 원하는 상태를 저장합니다. Scheduler가 Pod를 실행할 Node를 선택하면 해당 Worker Node의 kubelet이 Container Runtime을 통해 Container를 실행합니다. EKS가 Control Plane을 관리해도 Application 배포, Worker Capacity, Kubernetes Resource와 접근 권한은 사용자가 설계해야 합니다.

## 5. Data Plane 선택

---

| 방식 | Node 관리 | 특징 |
| --- | --- | --- |
| EC2 Managed Node Group | EKS가 Node Group의 생성·교체·Update를 지원 | EC2 Instance Type과 Node 설정을 제어할 수 있습니다. |
| Self-managed EC2 Node | 사용자가 Auto Scaling Group과 Node 수명주기를 관리 | Host 설정의 자유도가 높고 운영 범위도 넓습니다. |
| AWS Fargate | AWS가 Pod 실행 Host를 관리 | Node 접속이 없고 지원 기능과 Pod 구성에 제약이 있습니다. |

Managed Node Group에서도 Pod, DaemonSet, OS Image Update 시점과 Capacity 계획은 확인해야 합니다. Fargate는 Node 운영을 줄일 수 있지만 Host 접근, DaemonSet, Storage와 Network 요구 사항이 맞는지 먼저 검토합니다.

## 6. VPC와 Pod Network

---

일반적인 On-Premise Kubernetes는 Pod Network에 별도 CIDR을 두고 Overlay Network를 사용할 수 있습니다. EKS의 Amazon VPC CNI는 Pod에 VPC 주소를 할당해 VPC Resource와 직접 통신할 수 있게 합니다.

```text
VPC CIDR
├── Subnet A
│   ├── Worker Node A
│   └── Pod IP
└── Subnet B
    ├── Worker Node B
    └── Pod IP
```

VPC 주소를 사용한다고 해서 모든 통신이 자동 허용되는 것은 아닙니다. Route Table, Security Group, Network ACL, Kubernetes NetworkPolicy와 Pod가 사용하는 IAM 권한을 각각 확인합니다.

VPC와 Subnet의 범위는 다음과 같습니다.

| Resource | 범위 | 설계 기준 |
| --- | --- | --- |
| Region | AWS의 지리적 영역 | Cluster와 주 Resource를 배치합니다. |
| Availability Zone | Region 안의 독립된 위치 | 장애 분산을 위해 여러 AZ를 사용합니다. |
| VPC | Region 단위 Network | 다른 VPC와 기본적으로 격리됩니다. |
| Subnet | 하나의 Availability Zone | Public·Private Routing과 Workload 위치를 구분합니다. |

여러 Availability Zone에 Node를 배치하려면 AZ별 Subnet이 필요합니다. Public Subnet은 Internet Gateway로 향하는 Route를 가지며, Private Subnet은 Internet Gateway로 직접 Route하지 않습니다.

VPC는 하나의 Region에 속합니다. 다른 Region에 백업 환경을 만들려면 VPC와 관련 Resource를 해당 Region에 별도로 구성해야 합니다.

## 7. IAM 인증과 Kubernetes 인가

---

`kubectl` 요청은 AWS IAM 인증과 Kubernetes 인가를 모두 통과해야 합니다.

```text
AWS IAM Principal
  ↓ 임시 Credential로 인증
EKS Access Entry
  ↓ EKS Access Policy 또는 Kubernetes Group 연결
Kubernetes RBAC
  ↓ 허용된 API 작업 수행
Kubernetes Resource
```

새 Cluster에서는 IAM User와 장기 Access Key를 만드는 방식보다 IAM Identity Center나 AssumeRole 기반의 임시 Credential을 사용합니다. EKS Access Entry로 IAM Principal과 Cluster 권한을 연결하고 권한 범위를 Cluster 또는 Namespace 단위로 제한합니다.

`aws-auth` ConfigMap 기반 접근 관리는 Legacy 방식이며 EKS Cluster Access Management API로 대체되고 있습니다. 기존 Cluster를 즉시 변경하지는 않더라도 신규 Cluster에는 Access Entry 사용을 우선합니다.

## 8. ELB 연동

---

Cluster 외부에서 Pod로 접근하려면 Service, Ingress 또는 Gateway API Resource가 필요합니다.

| Kubernetes Resource | AWS 연동 | 용도 |
| --- | --- | --- |
| `Service` Type `LoadBalancer` | NLB 또는 Legacy CLB | TCP·UDP와 단순 Service 외부 노출 |
| `Ingress` | ALB | HTTP·HTTPS Host·Path Routing |
| `Gateway`·`Route` | 지원 Controller | 역할을 분리한 표준 Routing 구성 |

AWS Load Balancer Controller는 Service와 Ingress Resource를 감시하고 AWS Load Balancer를 생성합니다. Legacy Service Controller도 동작할 수 있지만 Critical Bug Fix 중심으로 유지되므로 새 Cluster는 AWS Load Balancer Controller 또는 EKS Auto Mode의 Load Balancing 기능을 사용합니다.

## 9. EKS 구축 도구

---

| 도구 | 역할 | 적합한 작업 |
| --- | --- | --- |
| AWS Console | Web UI에서 Resource 생성과 상태 확인 | 초기 학습과 상태 점검 |
| AWS CLI | EKS·IAM·CloudFormation API 호출 | Script와 반복 작업 |
| `eksctl` | VPC, EKS Cluster와 Node Group 구성 자동화 | 학습 환경과 EKS 중심 구축 |
| CloudFormation | AWS Resource를 Stack으로 선언 | Network와 공통 기반 Resource 관리 |

`eksctl create cluster`는 별도 VPC를 자동 생성할 수 있습니다. 기존 VPC의 Subnet을 전달하면 `eksctl`이 Route Table이나 NAT Gateway를 대신 구성하지 않으므로 사용자가 Subnet 요구 사항을 검증해야 합니다.

> **최종 정리**
> - CloudFormation은 Template으로 AWS Resource를 정의하고 Stack 단위로 상태를 관리합니다.
>
> - EKS는 Kubernetes Control Plane을 관리하며 사용자는 Data Plane과 Workload 구성을 선택합니다.
>
> - Control Plane의 배치 결정은 Worker Node의 kubelet과 Container Runtime 실행으로 이어집니다.
>
> - Pod가 VPC 주소를 사용해도 Route, Security Group과 Kubernetes 정책을 별도로 확인해야 합니다.
>
> - 사람의 Cluster 접근에는 장기 Access Key 대신 임시 Credential과 EKS Access Entry를 사용합니다.

## 참고 자료

---

- [CloudFormation Template 작성](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-guide.html)

- [Amazon EKS Cluster Access Management](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-access-management.html)

- [Amazon EKS Load Balancing](https://docs.aws.amazon.com/eks/latest/best-practices/load-balancing.html)

- [eksctl VPC 구성](https://docs.aws.amazon.com/eks/latest/eksctl/vpc-configuration.html)

다음 글에서는 [CloudFormation Network와 eksctl로 EKS Cluster 구축]({% post_url 2026-10-05-cloud-17-cloudformation-eksctl-cluster-nginx %})을 진행합니다.
