---
title: AWS 개요와 Global Infrastructure
description: AWS의 특징과 공동 책임 모델, 주요 서비스를 살펴보고 Region, Availability Zone, Edge Location으로 구성되는 Global Infrastructure를 정리합니다
date: 2026-09-28
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> AWS는 Compute, Network, Storage, Database와 운영 도구를 필요한 만큼 조합해 사용하는 Public Cloud입니다. 이 글에서는 AWS와 사용자의 책임 범위, 목적별 주요 Service, Region·Availability Zone·Edge Location의 차이와 AWS Resource를 조작하는 방법을 정리합니다.

## 1. AWS란 무엇인가

---

AWS(Amazon Web Services)는 Amazon이 제공하는 Public Cloud Platform입니다. 사용자는 물리 Server와 Data Center를 직접 구축하지 않고 Compute, Network, Storage, Database 등의 Resource를 필요할 때 생성하고 사용량에 따라 비용을 지불합니다.

AWS의 특징은 다음과 같습니다.

- **종량제**: Resource 종류, 사용 시간, 처리량과 저장 용량 등에 따라 비용이 계산됩니다.

- **Resource의 빠른 생성과 제거**: Management Console, AWS CLI, SDK와 IaC 도구로 Resource를 구성할 수 있습니다.

- **Service 조합**: EC2, VPC, S3, RDS, CloudWatch처럼 서로 다른 Service를 연결해 하나의 System을 구성합니다.

- **Global Infrastructure**: 여러 Region과 Availability Zone에 Workload를 배치할 수 있습니다.

- **Hybrid 구성**: VPN, Direct Connect, Transit Gateway 등을 이용해 AWS와 사내 Network를 연결할 수 있습니다.

- **규정 준수 지원**: AWS는 여러 Compliance Program과 인증 범위를 제공합니다. AWS가 인증을 보유하고 있다는 사실만으로 사용자가 구성한 Workload까지 자동으로 인증되는 것은 아니므로 공동 책임 모델과 적용 범위를 확인해야 합니다.

AWS Partner Network(APN)에 등록된 Partner를 통해 Migration, Architecture 설계와 운영 지원을 받을 수도 있습니다. Partner 등급과 회사 목록은 바뀔 수 있으므로 고정된 업체 수보다 [AWS Partner Finder](https://partners.amazonaws.com/)에서 현재 인증과 역량을 확인합니다.

### 1.1 공동 책임 모델

AWS가 모든 운영 작업을 대신 수행하는 것은 아닙니다. AWS와 사용자의 책임은 사용하는 Service에 따라 달라집니다.

| 구분 | AWS의 책임 | 사용자의 책임 |
| --- | --- | --- |
| EC2 | Data Center, 물리 Hardware, Hypervisor와 기반 Network | Guest OS, Package, Application, 계정, 보안 설정과 Data |
| RDS | 기반 Infrastructure와 Database Engine 운영 작업의 일부 | Schema, Account, Query, 접근 제어, Backup 정책 선택과 Data |
| S3 | Storage Infrastructure와 Service 가용성 | Bucket Policy, Object 권한, 암호화 선택, Lifecycle과 Data |

EC2를 단순히 Managed Service가 아니라고만 구분하면 책임 범위를 놓치기 쉽습니다. AWS는 물리 Infrastructure를 관리하고 사용자는 Instance 안의 OS와 Application을 관리합니다. RDS와 S3처럼 관리 범위가 더 넓은 Service는 사용자가 직접 수행할 운영 작업이 줄어듭니다.

## 2. AWS의 주요 Service

---

AWS Service의 수와 기능은 계속 변하므로 개수를 외우기보다 목적별로 구분하는 편이 좋습니다.

### 2.1 Compute와 Network

| Service | 역할 |
| --- | --- |
| Amazon EC2 | Virtual Server인 Instance를 제공합니다. |
| Amazon VPC | AWS Account 전용 Virtual Network를 구성합니다. |
| Elastic IP | Account에 할당하는 고정 Public IPv4 Address입니다. |
| Elastic Load Balancing | 여러 Target으로 Traffic을 분산합니다. |
| Amazon Route 53 | DNS와 Domain Routing 기능을 제공합니다. |
| AWS Certificate Manager | AWS Service에서 사용할 TLS Certificate를 발급하고 관리합니다. |
| Amazon CloudFront | Edge Location을 이용해 Content를 전달하는 CDN입니다. |
| Amazon API Gateway | HTTP, REST와 WebSocket API의 진입점을 관리합니다. |
| AWS Batch | Batch Job의 Compute 환경과 실행 Queue를 관리합니다. |
| AWS Transit Gateway | 여러 VPC와 On-premises Network의 연결을 중앙에서 관리합니다. |
| AWS Lambda | Server를 직접 관리하지 않고 Function 단위로 Code를 실행합니다. |

### 2.2 Container

| Service | 역할 |
| --- | --- |
| Amazon ECR | Container Image를 저장하는 Managed Registry입니다. |
| Amazon ECS | AWS가 제공하는 Container Orchestration Service입니다. |
| Amazon EKS | Managed Kubernetes Control Plane을 제공합니다. |

### 2.3 Storage와 Database

| Service | 역할 |
| --- | --- |
| Amazon S3 | Object Storage와 정적 Website Hosting을 제공합니다. 하나의 Object는 최대 5TB이며 큰 Object는 Multipart Upload를 사용합니다. |
| Amazon EBS | EC2에서 Disk처럼 연결해 사용하는 Block Storage입니다. |
| S3 Glacier Storage Class | 접근 빈도가 낮은 Data를 장기 보관하는 S3 Storage Class입니다. |
| Amazon RDS | 관계형 Database의 배포와 운영 작업을 관리합니다. |
| Amazon DynamoDB | Serverless Key-Value·Document Database입니다. |
| Amazon DocumentDB | MongoDB 호환 API를 제공하는 Document Database입니다. MongoDB의 모든 기능이 동일한 것은 아닙니다. |
| Amazon ElastiCache | Valkey, Memcached 또는 Redis OSS 호환 In-memory Cache를 제공합니다. |

S3 Glacier는 File을 압축하는 별도의 File Service가 아닙니다. S3 Object를 접근 빈도와 복구 시간에 맞는 Storage Class에 보관해 비용을 조절합니다.

### 2.4 Data, Messaging과 AI

| Service | 역할 |
| --- | --- |
| Amazon Redshift | Data Warehouse와 대규모 분석 Query를 제공합니다. |
| Amazon Athena | S3 Data를 SQL로 분석하는 Serverless Query Service입니다. |
| Amazon OpenSearch Service | 검색, Log 분석과 Monitoring에 사용하는 Managed Service입니다. |
| Amazon SQS | Application 사이의 Message Queue를 제공합니다. |
| Amazon SNS | Topic을 기준으로 Message를 여러 Subscriber에게 전달합니다. |
| Amazon SageMaker AI | Machine Learning Model의 구축, 학습과 배포를 지원합니다. |
| Amazon Bedrock | Foundation Model을 API로 사용하는 완전 관리형 생성형 AI Service입니다. |

`Amazon Elasticsearch Service`는 2021년에 `Amazon OpenSearch Service`로 이름이 변경되었습니다. 기존 Elasticsearch 호환 Domain과 과거 Billing 자료에서는 이전 이름이 남아 있을 수 있습니다.

Bedrock에서 사용할 수 있는 Model Provider와 Model은 Region과 시점에 따라 달라집니다. Anthropic, Meta, Mistral AI와 Amazon 등의 Model을 사용하기 전에 대상 Region의 Model Access와 가격을 확인합니다.

### 2.5 IoT와 End User Computing

| Service | 역할 |
| --- | --- |
| AWS IoT Core | Device와 Cloud Application 사이의 Message 통신과 Device 연결을 관리합니다. |
| FreeRTOS | Microcontroller용 Open Source Real-time Operating System입니다. |
| Amazon WorkSpaces | Managed Virtual Desktop을 제공합니다. |

### 2.6 개발과 운영 도구

| Service | 역할 |
| --- | --- |
| AWS CloudFormation | Template으로 AWS Resource를 선언하고 배포하는 IaC Service입니다. |
| AWS CodeBuild | Source Code를 Compile하고 Test하며 Artifact를 생성합니다. |
| AWS CodeCommit | AWS가 관리하는 Git Repository입니다. |
| AWS CodeDeploy | EC2, Lambda와 ECS 등에 Application을 배포합니다. |
| AWS CodePipeline | Source, Build, Test와 Deploy Stage를 Pipeline으로 연결합니다. |
| Amazon CloudWatch | Metric, Log, Alarm과 Dashboard를 제공합니다. |
| AWS Cost Explorer | 사용 비용을 탐색하고 분석합니다. |
| AWS Budgets | 비용 또는 사용량 Threshold에 대한 예산과 알림을 구성합니다. |
| AWS Cost and Usage Report | 비용과 사용량의 상세 Record를 S3에 전달합니다. |
| AWS Compute Optimizer | Resource 사용량을 분석해 적정 크기를 추천합니다. |
| AWS Trusted Advisor | 비용, 성능, 보안, 내결함성과 Service Quota를 점검합니다. |

CloudFormation은 AWS Resource Provisioning에 초점을 둡니다. Terraform은 여러 Provider의 Infrastructure를 선언적으로 관리하며, Ansible은 생성된 Server의 Package와 설정 상태를 관리하는 데 주로 사용합니다. 세 도구의 범위가 일부 겹치더라도 같은 역할로 보기는 어렵습니다.

### 2.7 현재 상태가 변경된 Service

| 이전 자료의 Service | 2026년 9월 기준 상태 |
| --- | --- |
| AWS Cloud9 | 2024년 7월 25일부터 신규 고객에게 제공되지 않습니다. 기존 고객은 계속 사용할 수 있으며 AWS CloudShell과 IDE용 AWS Toolkit을 대안으로 검토합니다. |
| AWS CodeStar Project | 2024년 7월 31일 Project 생성과 조회 지원이 종료되었습니다. CodeStar가 생성한 기존 Resource와 AWS CodeConnections는 별도로 동작합니다. |
| Amazon CodeCatalyst | 2025년 11월 7일부터 신규 고객에게 제공되지 않으며 기존 고객도 새 Space를 만들 수 없습니다. 신규 구성에서는 CodeBuild, CodePipeline, CodeDeploy, CodeArtifact 또는 외부 DevOps Platform을 검토합니다. |
| AWS IoT Things Graph | 2022년 11월 9일 지원이 종료되었습니다. |
| Amazon Elasticsearch Service | 현재 이름은 Amazon OpenSearch Service입니다. |

AWS CodeCommit은 2024년에 신규 고객 등록이 한때 제한되었으나 2025년 11월 25일부터 신규 고객에게 다시 제공되고 있습니다. Service 상태는 변경될 수 있으므로 새 Architecture를 설계할 때 공식 Service 문서를 확인합니다.

## 3. Region, Availability Zone과 Edge Location

---

### 3.1 Region

Region은 AWS Infrastructure가 배치된 독립적인 지리 영역입니다. 서울 Region의 Code는 `ap-northeast-2`입니다.

Region을 선택할 때는 다음 항목을 확인합니다.

- 사용자와 가까운 위치인지 확인합니다.

- 필요한 Service와 Instance Type이 제공되는지 확인합니다.

- Data Residency와 Compliance 요구사항을 충족하는지 확인합니다.

- Service 가격과 다른 Region으로 전송되는 Data Transfer 비용을 확인합니다.

Resource는 대부분 Region 단위로 생성됩니다. EC2 Key Pair와 Elastic IP도 Region에 속하므로 다른 Region에서 그대로 사용할 수 없습니다.

### 3.2 Availability Zone

Availability Zone(AZ)은 Region 안에서 전력, Network와 물리 장애 영역을 분리한 하나 이상의 Data Center 집합입니다. AZ를 Data Center 한 개와 동일한 개념으로 단정하지 않습니다.

서울 Region은 2020년부터 4개의 AZ를 제공합니다. 다음 AWS CLI 명령은 선택한 Account에서 사용할 수 있는 AZ 이름과 AZ ID를 확인합니다.

```bash
aws ec2 describe-availability-zones \
  --region ap-northeast-2 \
  --query 'AvailabilityZones[].{Name:ZoneName,ID:ZoneId,State:State}' \
  --output table
```

여러 AZ에 EC2 Instance를 분산하고 Load Balancer로 Traffic을 전달하면 한 AZ에 장애가 발생했을 때 다른 AZ의 Instance가 요청을 처리할 수 있습니다.

### 3.3 Edge Location

Edge Location은 CloudFront와 Route 53 같은 Service가 사용자와 가까운 곳에서 요청을 처리하도록 구성한 Point of Presence입니다. Region이나 AZ처럼 Application Server를 일반 EC2 Instance로 배치하는 위치가 아닙니다.

```text
사용자
  ↓
가까운 Edge Location
  ↓ Cache Miss 또는 Origin 요청
Region의 Load Balancer 또는 Application
```

## 4. AWS를 조작하는 방법

---

### 4.1 Management Console

Management Console은 Web Browser에서 AWS Resource를 관리하는 GUI입니다. Service별 Dashboard에서 Resource를 생성하고 상태를 확인할 수 있습니다.

Console 작업을 시작하기 전에 오른쪽 위의 Region 표시를 확인합니다. Resource가 보이지 않을 때는 다른 Region을 선택했는지 먼저 확인합니다.

### 4.2 AWS CLI

AWS CLI는 Terminal과 Script에서 AWS API를 호출합니다. 반복 작업과 자동화에는 Console보다 CLI가 적합합니다.

장기 Access Key를 Local File에 저장하는 방식보다 IAM Identity Center를 이용한 SSO Profile을 우선 사용합니다.

```bash
aws configure sso
```

Login 후 현재 호출 주체를 확인합니다.

```bash
aws sso login --profile <PROFILE_NAME>
aws sts get-caller-identity --profile <PROFILE_NAME>
```

`aws configure`로 Access Key를 입력해야 한다면 Root User의 Key를 만들지 않고 최소 권한을 가진 IAM Identity를 사용합니다. Credential File은 Version Control에 포함하지 않습니다.

다음 글인 [Amazon EC2와 ELB 기반 고가용성 구성](/cloud-05-aws-ec2-elb-high-availability/)에서는 EC2 Instance를 생성하고 Security Group, ALB와 Auto Scaling Group을 연결합니다.

## 참고 자료

---

- [AWS Region과 Availability Zone](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-regions.html)

- [AWS Cloud9 현재 상태](https://docs.aws.amazon.com/cloud9/latest/user-guide/history.html)

- [AWS CodeCommit 변경 이력](https://docs.aws.amazon.com/codecommit/latest/userguide/history.html)

- [AWS CodeStar Project 지원 변경](https://docs.aws.amazon.com/codestar/latest/userguide/create-github.html)

- [Amazon CodeCatalyst 현재 상태](https://docs.aws.amazon.com/codecatalyst/latest/userguide/migration.html)

- [지원이 종료된 AWS Service](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html)

- [Amazon OpenSearch Service 이름 변경](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/rename.html)
