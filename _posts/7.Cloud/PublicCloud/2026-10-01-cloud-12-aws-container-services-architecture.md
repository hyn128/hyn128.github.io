---
title: AWS Container Service와 ECS·EKS 실행 구조
description: ECS·EKS Control Plane과 EC2·Fargate Data Plane의 역할을 구분하고 비용·확장성·운영 책임에 따라 Container 실행 구조를 선택합니다
date: 2026-10-01
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> AWS에서 Container를 실행하려면 Image를 보관하는 Registry, 배치 상태를 관리하는 Orchestrator와 실제 Workload가 실행되는 Compute 환경을 함께 선택해야 합니다. ECS와 EKS는 Container 배치를 제어하고, EC2와 Fargate는 Task 또는 Pod가 실행될 Compute를 제공합니다. 이 글에서는 각 구성 요소의 책임 범위와 네 가지 대표 조합을 비교합니다.

## 1. Container 도입 전에 확인할 사항

---

Container는 Application과 Library를 Image에 묶어 실행 환경의 일관성을 높입니다. 여러 Container가 Network로 통신하는 구조가 되면 분산 시스템의 특성도 함께 나타납니다.

- Container 내부 File System은 교체 시 사라질 수 있으므로 영구 Data를 외부 Storage나 Managed Database에 보관합니다.

- Service 사이의 결합도를 낮추고 Timeout, Retry와 장애 격리를 설계합니다.

- Image Build부터 Registry Push, 배포와 Rollback까지의 흐름을 관리합니다.

- Image 취약점, Runtime 권한, Secret과 Network 경계를 통제합니다.

- Log와 Metric을 Container 외부로 전송해 교체 이후에도 조사할 수 있게 합니다.

The Twelve-Factor App은 환경별 설정 분리, Stateless Process, Log Stream과 빠른 시작·종료처럼 Cloud Application을 설계할 때 참고할 원칙을 제공합니다. 모든 항목을 기계적으로 적용하는 규칙은 아니며 Workload 특성에 맞게 선택합니다.

## 2. AWS Container 구성 요소

---

AWS Container 환경은 다음 세 영역으로 나누어 이해할 수 있습니다.

```text
Source Code
  ↓ Build
Container Image
  ↓ Push
Amazon ECR
  ↓ Pull
ECS 또는 EKS Control Plane
  ↓ 배치 결정
EC2 또는 Fargate Compute
  ↓
Task 또는 Pod 실행
```

| 영역 | AWS Service | 역할 |
| --- | --- | --- |
| Image Registry | Amazon ECR | Image 저장, Version·Lifecycle과 취약점 Scan 관리 |
| Orchestration | Amazon ECS | AWS 고유 Task·Service Model로 Container 배치 관리 |
| Orchestration | Amazon EKS | AWS가 Kubernetes Control Plane을 관리 |
| Compute | Amazon EC2 | 사용자가 Instance와 OS를 관리하는 실행 환경 |
| Compute | AWS Fargate | Host를 직접 관리하지 않는 Serverless Container Compute |

ECS와 EKS를 단순히 Control Plane Service, EC2와 Fargate를 Data Plane 선택지로 구분하면 구조를 이해하기 쉽습니다. 다만 실제 서비스 경계는 더 넓습니다. ECS는 Scheduling, Service Desired Count와 Deployment를 관리하고, EKS에서는 Kubernetes API와 Controller가 Pod의 원하는 상태를 관리합니다.

## 3. Amazon ECS

---

Amazon Elastic Container Service(ECS)는 AWS가 제공하는 Managed Container Orchestrator입니다.

주요 Resource는 다음과 같습니다.

| Resource | 역할 |
| --- | --- |
| Cluster | Service와 Task를 논리적으로 묶는 경계 |
| Task Definition | Image, CPU·Memory, Port, Environment와 IAM Role 정의 |
| Task | Task Definition으로 실행된 Workload Instance |
| Service | 원하는 Task 수를 유지하고 Deployment와 Load Balancer 연결 관리 |
| Capacity Provider | Task가 사용할 Fargate 또는 EC2 Capacity와 Scaling 전략 정의 |

ECS 자체가 Application Container를 실행하는 Host는 아닙니다. Scheduler가 배치할 위치를 결정하고 실제 Task는 Fargate 또는 ECS Cluster에 등록된 EC2 Capacity에서 실행됩니다.

현재 ECS는 Fargate·Fargate Spot, EC2 Auto Scaling Group, ECS Managed Instances와 External Instance 같은 Capacity 선택지를 제공합니다. 이 글에서는 기본 구조를 비교하기 위해 EC2와 Fargate에 집중합니다.

## 4. Amazon EKS

---

Amazon Elastic Kubernetes Service(EKS)는 Managed Kubernetes Service입니다. AWS가 Kubernetes API Server와 etcd를 포함한 Control Plane의 가용성과 운영을 담당합니다.

```text
Developer 또는 CI/CD
  ↓ Kubernetes API
EKS Control Plane
  ↓ Scheduling·Desired State
EC2 Node Group 또는 Fargate Profile
  ↓ kubelet 또는 Fargate Runtime
Pod 실행
```

Managed Control Plane을 사용해도 다음 책임은 사용자에게 남습니다.

- Kubernetes Version Upgrade 계획과 API Compatibility 확인

- Node Group 또는 Fargate Profile 구성

- Core Add-on, Network, Load Balancer Controller와 Storage Driver 관리

- Workload Manifest, Resource Request·Limit과 Pod Security 설정

- Application, Cluster Event와 Workload Log Monitoring

EKS Kubernetes Minor Version은 Standard Support 14개월 이후 Extended Support 12개월을 제공합니다. Extended Support에는 추가 비용이 발생하므로 Version별 종료 시점을 확인하고 Upgrade를 계획합니다.

## 5. EC2를 실행 환경으로 사용할 때

---

EC2를 ECS Container Instance 또는 EKS Node로 사용하면 Instance Type, OS, Kernel, Storage와 Network 구성을 세밀하게 선택할 수 있습니다.

| 장점 | 운영 책임 |
| --- | --- |
| GPU, 큰 Memory와 특수 Instance 사용 | Instance Provisioning과 Capacity 계획 |
| Daemon, Privileged Workload와 Host 설정 제어 | OS Patch와 Container Runtime 관리 |
| Image Cache와 Instance 단위 비용 최적화 | Auto Scaling, 장애 Node 교체와 Monitoring |
| EBS Instance Store 등 Storage 선택 폭 | 보안 설정과 AMI Lifecycle 관리 |

EC2 Instance가 준비되어 있어도 Task 또는 Pod의 Resource Request를 수용할 여유가 없으면 Scheduling에 실패합니다. Workload Scaling과 Node Scaling을 함께 설계해야 합니다.

## 6. Fargate를 실행 환경으로 사용할 때

---

AWS Fargate는 ECS Task 또는 EKS Pod를 실행할 Host를 AWS가 Provisioning하고 관리하는 Serverless Container Compute입니다.

사용자는 다음 값을 중심으로 정의합니다.

- Container Image와 실행 Command

- vCPU와 Memory 조합

- Network, Security Group과 Subnet

- IAM Role, Environment Variable과 Secret

- Ephemeral Storage와 영구 Storage 연결

Host OS Patch와 Cluster Capacity 확보를 직접 수행하지 않는다는 장점이 있지만 Host Kernel을 조정하거나 Host에 접근하는 Workload에는 적합하지 않습니다. Fargate가 항상 EC2보다 비싸거나 TCO가 항상 낮다고 단정할 수 없습니다. Task 이용률, EC2 예약 할인, 운영 인력, Scaling Pattern과 중단 허용 여부를 함께 계산합니다.

ECS Fargate의 Linux Task Size는 현재 0.25 vCPU부터 32 vCPU까지의 정해진 CPU·Memory 조합을 제공합니다. 32 vCPU Task는 60 GiB, 120 GiB 또는 244 GiB Memory를 선택할 수 있습니다. 지원 조합과 Platform Version은 변경될 수 있으므로 배포 Region의 최신 표를 확인합니다. 기본 Ephemeral Storage는 20 GiB이며 21~200 GiB 범위로 확장할 수 있습니다.

Linux Task와 Pod의 사용량은 Image Download 시작부터 종료까지 초 단위로 계산되며 최소 1분이 청구됩니다. vCPU, Memory, 추가 Ephemeral Storage뿐 아니라 Public IPv4, Data Transfer와 CloudWatch 같은 연계 Service 비용도 합산합니다.

> **주의**
> Fargate를 사용하는 ECS Task와 EKS Pod의 지원 기능은 같지 않습니다. EKS Fargate는 DaemonSet, Privileged Container와 GPU를 지원하지 않으며 Pod별로 격리된 Compute 경계에서 실행됩니다. EBS Volume은 Fargate Pod에 Mount할 수 없고 정적으로 Provisioning한 EFS를 사용할 수 있습니다.

## 7. Amazon ECR

---

Amazon Elastic Container Registry(ECR)는 Private·Public Container Image Repository를 제공하는 Managed Registry입니다.

```text
Developer 또는 CI Runner
  ↓ Authenticate
Docker Build
  ↓ Tag
ECR Repository
  ↓ Pull
ECS Task 또는 EKS Pod
```

Repository 운영 시 다음 항목을 관리합니다.

| 항목 | 목적 |
| --- | --- |
| Immutable Tag | 배포된 Tag가 다른 Image로 덮어써지는 문제 방지 |
| Lifecycle Policy | 오래된 Image와 Untagged Image 정리 |
| Image Scan | OS와 Package 취약점 확인 |
| Encryption | 저장 Image 암호화 |
| Repository Policy·IAM | Push·Pull 주체 최소 권한 부여 |

`latest`만 배포 기준으로 사용하면 실행 중인 Revision을 추적하기 어렵습니다. Commit SHA나 Release Version처럼 변경되지 않는 Tag를 함께 사용합니다.

## 8. ECS on EC2

---

ECS on EC2는 ECS가 Task를 배치하고 사용자가 관리하는 EC2 Instance에서 Container가 실행되는 구조입니다.

```text
ECS Service
  ↓ Task 배치
EC2 Auto Scaling Group
  ├── ECS Task A
  ├── ECS Task B
  └── ECS Agent
```

| 관점 | 특성 |
| --- | --- |
| 비용 | Instance 단위 구매이므로 이용률을 높이면 효율적이지만 유휴 Capacity 비용이 발생 |
| 확장 | Task Scaling과 EC2 Capacity Scaling을 함께 구성 |
| 운영 | AMI, Patch, ECS Agent와 장애 Instance 교체 관리 필요 |
| 유연성 | GPU, 특수 Instance, Host 설정과 Storage 선택 폭이 넓음 |
| 조사 | Systems Manager Session Manager, ECS Exec와 Log를 이용 가능 |

Host에 직접 SSH하는 방식은 Key와 Inbound Port 관리 부담이 있습니다. 운영 조사에는 Systems Manager Session Manager나 ECS Exec를 우선 검토합니다.

## 9. ECS on Fargate

---

ECS on Fargate는 ECS Service가 원하는 Task 수를 관리하고 Fargate가 각 Task의 Compute를 제공합니다.

```text
ECS Service
  ↓ Desired Count
Fargate
  ├── Task A + ENI
  ├── Task B + ENI
  └── Task C + ENI
```

| 관점 | 특성 |
| --- | --- |
| 비용 | Task에 할당한 vCPU·Memory·Storage와 실행 시간 기준 과금 |
| 확장 | Host Capacity를 먼저 확보하지 않고 Task 수 확장 가능 |
| 운영 | Host OS와 Container Runtime을 AWS가 관리 |
| 제약 | 지원 CPU·Memory 조합, Host 접근과 Kernel 조정 제한 |
| Network | `awsvpc` Mode로 Task마다 ENI와 Security Group 적용 |

Image Pull과 ENI 준비가 필요하므로 새 Task 시작 시간이 Workload와 Network 상태에 따라 달라집니다. 필요 Capacity를 항상 즉시 확보할 수 있다고 가정하지 않고 Service Auto Scaling, Health Check와 Deployment 설정을 검증합니다.

## 10. EKS on EC2

---

EKS on EC2는 AWS가 Control Plane을 관리하고 사용자가 Managed Node Group, Self-managed Node 또는 Auto Mode 등으로 Worker Capacity를 제공합니다.

| 관점 | 특성 |
| --- | --- |
| 비용 | EKS Cluster 요금과 EC2·EBS·Load Balancer 등 Resource 비용 발생 |
| 확장 | Pod Autoscaling과 Node Scaling을 함께 설계 |
| 운영 | Kubernetes와 Node OS·AMI Upgrade 책임이 함께 존재 |
| 유연성 | DaemonSet, GPU, Privileged Workload와 다양한 CSI Driver 사용 가능 |
| 생태계 | Kubernetes API와 Tool을 활용할 수 있으나 학습·운영 범위가 넓음 |

AWS는 EKS Control Plane을 관리하지만 사용자가 설치한 Controller와 Workload의 오류까지 자동으로 해결하지는 않습니다. Pod 상태, Event, Node Condition과 Add-on Compatibility를 직접 확인해야 합니다.

## 11. EKS on Fargate

---

EKS on Fargate는 Fargate Profile의 Namespace와 Label Selector에 일치하는 Pod를 Fargate에서 실행합니다.

```text
Pod 생성 요청
  ↓
EKS Scheduler
  ↓ Fargate Profile 일치
Fargate Compute 준비
  ↓
Pod 실행
```

하나의 Fargate Pod는 전용 Compute 경계를 사용하며 여러 Container가 포함된 Pod라면 그 Container들은 같은 경계 안에서 실행됩니다. 일반 EC2 Node처럼 사용자가 Node에 로그인하거나 여러 Pod를 직접 Packing하도록 관리하지 않습니다.

Fargate가 Worker Host 운영을 줄여도 EKS Control Plane Version과 Kubernetes Add-on Upgrade는 계속 관리해야 합니다. Fargate Host OS를 직접 Upgrade하지 않는다는 뜻과 EKS Cluster Upgrade가 필요 없다는 뜻을 구분합니다.

외부 Traffic은 Service와 Ingress 또는 Gateway API Resource로 선언하고 AWS Load Balancer Controller가 실제 Load Balancer를 구성합니다. Resource 종류만으로 ALB·NLB가 자동 결정되는 것은 아니며 Controller Version과 Annotation 또는 지원 필드를 확인합니다.

다음 Workload에는 EC2 Node를 검토합니다.

- DaemonSet이 필요한 Node Agent

- Privileged Container나 Host Network·Host Port가 필요한 Workload

- GPU Workload

- EBS Volume을 직접 Mount해야 하는 Pod

- 특정 Kernel, Device 또는 Host 설정이 필요한 Workload

## 12. ECS·EKS와 EC2·Fargate 비교

---

Architecture는 Service 사용료만으로 선택하지 않습니다.

| 판단 기준 | 확인할 내용 |
| --- | --- |
| 비용 | Compute·Storage·Network 요금, 유휴 Capacity, 운영 인력과 교육 비용 |
| 확장성 | 배포 시작 시간, 수평 Scaling, 개별 Task·Pod의 수직 Resource 범위 |
| 신뢰성 | 장애 영역, 자동 복구, 조사 수단, Backup·Rollback과 지원 범위 |
| 운영성 | Patch, Version Upgrade, Image Lifecycle, Observability와 권한 관리 |
| 인력 | ECS·AWS 운영 경험 또는 Kubernetes 전문성 확보 가능 여부 |

| 조합 | 적합한 경우 | 주요 부담·제약 |
| --- | --- | --- |
| ECS on EC2 | AWS 중심 운영, GPU·특수 Instance, 높은 지속 이용률 | EC2 Capacity와 OS 운영 |
| ECS on Fargate | AWS 중심 운영, Host 관리 최소화, 독립적인 Task Scaling | 지원 사양과 Host 기능 제한 |
| EKS on EC2 | Kubernetes API·생태계, 복잡한 Workload와 Node 제어 | Kubernetes와 Node 운영 범위 |
| EKS on Fargate | Kubernetes Pod 중 Host 기능이 필요 없는 일부 Workload | DaemonSet·GPU·Privileged·EBS 제약 |

Control Plane과 Compute를 선택할 때 다음 순서로 판단합니다.

1. Kubernetes API와 생태계가 필수인지 결정합니다.

2. GPU, DaemonSet, Privileged Mode, Host Tuning과 Storage 요구를 확인합니다.

3. Workload가 지속형인지 변동형인지 확인하고 Compute 비용을 계산합니다.

4. Cluster와 Host 운영에 투입할 인력과 Upgrade 주기를 확인합니다.

5. 장애 복구, 배포 속도, 관측성과 Support 요구를 검증합니다.

## 13. 관련 Compute Service

---

### 13.1 AWS Lambda

Lambda는 Event에 따라 Function Code나 Lambda 호환 Container Image를 실행하는 Serverless Compute입니다. 기본 Lambda 실행 환경은 Firecracker MicroVM 기술로 격리되며 사용자가 ECS·EKS Cluster나 Worker Node를 구성하지 않습니다.

Function Code, Memory·Timeout, IAM Execution Role과 Trigger를 중심으로 관리합니다. 짧은 Event 처리, API Handler와 Schedule 작업에 적합하지만 실행 시간, Runtime, Storage와 Network Model이 일반 Container Service와 다릅니다. Container Image를 지원한다는 이유만으로 ECS Task와 동일한 Runtime으로 해석하지 않습니다.

### 13.2 AWS App Runner

App Runner는 Source Repository 또는 Container Image에서 Web Application을 Build·배포하고 Network, Load Balancing과 Scaling을 관리하는 Service입니다.

현재 AWS는 App Runner를 신규 고객에게 제공하지 않으며 기존 고객만 새 Resource와 Service를 계속 만들 수 있습니다. 신규 기능도 계획되어 있지 않습니다. AWS는 기존 App Runner Workload의 대안으로 ECS Express Mode를 안내합니다. 따라서 신규 Architecture의 기본 선택지로 App Runner를 채택하기 전에 Account 사용 가능 여부와 Migration 방향을 확인합니다.

## 14. 적용 사례

---

### 14.1 GPU 또는 Machine Learning Workload

Fargate는 GPU를 지원하지 않으므로 GPU Instance를 사용하는 ECS on EC2 또는 EKS on EC2를 선택합니다. Training Framework, Scheduling과 Kubernetes 생태계가 필요하지 않다면 ECS가 단순할 수 있습니다.

### 14.2 큰 Local Disk가 필요한 Full Node

Blockchain Network의 Node는 역할에 따라 Full Node, Validator와 Miner 등으로 나뉩니다. 전체 Block Data를 보관하는 Full Node처럼 큰 Storage와 Host Tuning이 필요한 Workload는 EC2 기반 구성이 적합할 수 있습니다. EBS Volume 크기는 유한하므로 무제한 Storage로 표현하지 않고 증가 한도, Throughput, Snapshot과 복구 시간을 설계합니다.

### 14.3 변동이 큰 Stateless API

Host 운영을 줄이고 Task 단위로 확장하려면 ECS on Fargate를 우선 검토할 수 있습니다. 요청량, 최소 Task 수, Cold Start 영향과 Database Connection 수를 함께 확인합니다.

### 14.4 Kubernetes 표준 API가 필요한 Platform

여러 팀이 Kubernetes Resource와 Operator를 사용하거나 Cloud 사이 이식성이 중요한 경우 EKS를 검토합니다. Kubernetes 사용 자체가 이식성을 자동으로 보장하지는 않으며 AWS Load Balancer, IAM과 Storage 연동은 AWS 종속성을 만듭니다.

### 14.5 Self-managed Kubernetes가 필요한 환경

On-premises 또는 EC2에서 Control Plane과 Worker를 모두 직접 운영하면 Distribution, Network와 보안 구성을 가장 넓게 제어할 수 있습니다. 반면 etcd Backup, Control Plane 고가용성, Certificate와 Version Upgrade 책임도 모두 운영 팀에 있습니다. 규제, 기존 Platform 제약이나 명확한 비용 근거가 있을 때 선택합니다.

### 14.6 높은 Resource 이용률이 필요한 환경

지속적으로 실행되는 Task를 EC2에 조밀하게 배치하면 Instance 비용을 효율적으로 사용할 수 있습니다. 변동이 큰 Workload는 Fargate에서 필요한 vCPU·Memory를 Task 단위로 할당하는 편이 운영상 단순할 수 있습니다. CPU와 Memory를 100%에 가깝게 사용하는 것 자체를 목표로 삼지 않고 응답 지연, Throttling, 장애 여유와 Scaling 시간을 함께 측정합니다.

> **최종 정리**
> - ECR은 Image를 저장하고 ECS·EKS는 원하는 Container 상태와 배치를 관리합니다.
>
> - EC2는 Host 제어와 선택 폭이 넓지만 Capacity, OS와 Patch를 사용자가 운영합니다.
>
> - Fargate는 Host 운영을 줄이지만 지원 사양과 Host 기능에 제약이 있습니다.
>
> - 비용, 확장성, 신뢰성과 운영 인력을 함께 비교해야 하며 하나의 조합이 모든 Workload에 항상 유리하지는 않습니다.

## 참고 자료

---

- [Amazon ECS Launch Type과 Capacity Provider 비교](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/capacity-launch-type-comparison.html)

- [ECS Fargate Task 정의](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-tasks-services.html)

- [ECS Fargate Ephemeral Storage](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-storage.html)

- [AWS Fargate 요금과 지원 Resource 조합](https://aws.amazon.com/fargate/pricing/)

- [Amazon EKS의 AWS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)

- [Amazon EKS Kubernetes Version](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)

- [Amazon ECR 개념](https://docs.aws.amazon.com/AmazonECR/latest/userguide/concept-and-components.html)

- [AWS App Runner Availability 변경](https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html)

- [AWS Lambda 실행 환경 Lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)

- [The Twelve-Factor App](https://12factor.net/)

다음 글인 [Django REST API 생성과 Container 배포 준비](/cloud-13-django-rest-api-container-preparation/)에서는 Python 가상환경, Django Project·Application과 REST Endpoint를 구성합니다.
