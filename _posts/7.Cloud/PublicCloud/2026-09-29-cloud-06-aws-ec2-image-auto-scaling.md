---
title: EC2 Snapshot·AMI와 Auto Scaling 실습
description: EBS Snapshot과 AMI의 관계를 확인하고 Launch Template, Target Group과 Auto Scaling Group을 연결해 EC2 Instance의 생성·교체·확장을 자동화합니다
date: 2026-09-29
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> Auto Scaling Group은 Instance 수만 조절하는 기능이 아닙니다. 어떤 Image와 설정으로 Instance를 만들지 Launch Template에 기록하고, 여러 Availability Zone과 Target Group을 연결해야 동일한 Server를 반복 생성할 수 있습니다. 이 글에서는 EBS Snapshot과 AMI를 준비한 뒤 Launch Template과 Auto Scaling Group을 구성하고, Instance 교체와 Scaling 동작을 검증합니다.

## 1. 구성 요소의 연결 관계

---

Auto Scaling 실습에는 다음 Resource가 사용됩니다.

| Resource | 역할 |
| --- | --- |
| EBS Snapshot | 특정 시점의 EBS Volume Block을 보존합니다. |
| AMI | Instance를 시작하는 데 필요한 Root Volume Template, Block Device Mapping과 권한 정보를 가집니다. |
| Launch Template | AMI, Instance Type, Key Pair, Security Group, Storage와 User Data를 Version별로 저장합니다. |
| Auto Scaling Group | Minimum, Desired, Maximum Capacity와 선택한 Subnet에 따라 Instance를 유지합니다. |
| Target Group | ALB가 요청을 전달할 Instance와 Health Check 상태를 관리합니다. |
| Scaling Policy | Metric, 일정 또는 예측 결과에 따라 Desired Capacity를 변경합니다. |

```text
EBS Volume
    ↓ Snapshot
Custom AMI
    ↓
Launch Template
    ↓
Auto Scaling Group
    ├── AZ A의 EC2 Instance
    └── AZ B의 EC2 Instance
              ↓ 자동 등록
          Target Group
              ↓
             ALB
```

Control 영역에서는 Launch Template과 Auto Scaling Group이 원하는 Instance 구성을 정의합니다. 실제 Data Plane에서는 각 Availability Zone에 생성된 Instance가 Application을 실행하고 Target Group을 통해 Traffic을 받습니다.

## 2. EBS Snapshot과 AMI

---

### 2.1 EBS Snapshot

EBS Snapshot은 Volume의 특정 시점 데이터를 보존합니다. 첫 Snapshot 이후의 Snapshot은 이전 Snapshot 이후 변경된 Block만 추가로 저장하는 증분 방식입니다. Snapshot 하나를 삭제하더라도 다른 Snapshot을 복원하는 데 필요한 Block은 유지됩니다.

Snapshot은 AWS가 관리하는 Amazon S3 기반 Storage에 저장되지만 사용자의 S3 Bucket에서 File처럼 조회하거나 Download하는 형태는 아닙니다. 비용은 Snapshot에 저장된 Data 양과 Archive, 복원 같은 추가 기능 사용량에 따라 발생합니다.

Snapshot은 다음 작업에 사용합니다.

- Snapshot으로 새로운 EBS Volume을 생성해 원본 Volume을 복제합니다.

- 장애나 잘못된 변경이 발생했을 때 별도의 Volume으로 복원합니다.

- EBS-backed AMI가 참조할 Root Volume과 추가 Volume Snapshot을 만듭니다.

- Amazon Data Lifecycle Manager 정책으로 Snapshot 생성, 보존과 삭제를 자동화합니다.

### 2.2 AMI

AMI(Amazon Machine Image)는 EC2 Instance를 시작하는 Template입니다. EBS-backed AMI에는 다음 정보가 포함됩니다.

- OS, Package, Application과 설정이 들어 있는 EBS Snapshot입니다.

- Instance 시작 시 연결할 Volume을 정의하는 Block Device Mapping입니다.

- AMI를 사용할 수 있는 AWS Account를 결정하는 Launch Permission입니다.

| 구분 | EBS Snapshot | AMI |
| --- | --- | --- |
| 기준 | EBS Volume | EC2 Instance 시작 구성 |
| 목적 | Volume Data Backup과 복원 | 동일한 구성의 Instance 생성 |
| 결과 | 새 EBS Volume | 새 EC2 Instance |
| 부팅 정보 | 직접 제공하지 않음 | Root Device와 Architecture 등 시작 정보 포함 |

AMI를 삭제할 때는 AMI를 Deregister한 뒤 연결된 Snapshot의 보존 여부도 확인해야 합니다. AMI 등록을 해제해도 Snapshot이 자동으로 모두 삭제된다고 가정하면 안 됩니다.

## 3. 기준 Instance 준비와 AMI 생성

---

### 3.1 기준 Instance 점검

AMI를 만들 Instance에서 Application이 정상적으로 실행되는지 먼저 확인합니다.

```bash
systemctl is-active apache2
curl -fsS http://127.0.0.1/
lsblk -f
```

Application이 File이나 Database에 계속 쓰고 있다면 AMI 생성 중 Data가 일관되지 않을 수 있습니다. 운영 Data를 포함한 Image는 Application을 중지하거나 Flush하고, 필요한 경우 별도의 Backup 절차를 수행한 뒤 생성합니다.

### 3.2 Management Console에서 AMI 생성

`[EC2] → [인스턴스]`에서 기준 Instance를 선택하고 다음 순서로 진행합니다.

1. `[작업] → [이미지 및 템플릿] → [이미지 생성]`을 선택합니다.

2. Image 이름과 설명을 입력합니다.

3. Root Volume과 추가 EBS Volume의 크기, 유형과 종료 시 삭제 여부를 확인합니다.

4. Application 일관성이 필요한 경우 기본 Reboot 동작을 유지합니다. `No reboot`는 중단 시간을 줄이지만 File System과 Application 상태를 사용자가 보장해야 합니다.

5. Image를 생성한 뒤 `[AMI]`와 `[스냅샷]` 화면에서 상태가 `available`, `completed`인지 확인합니다.

AMI가 준비되기 전에는 Launch Template에서 사용할 수 없습니다.

## 4. Auto Scaling용 Security Group

---

HTTP와 SSH를 모두 `0.0.0.0/0`에 허용하는 Security Group은 사용하지 않습니다. ALB와 EC2의 역할에 따라 Group을 분리합니다.

| Security Group | Inbound Rule | 목적 |
| --- | --- | --- |
| ALB Security Group | TCP 80 또는 443, Source `0.0.0.0/0` | Client 요청을 받습니다. |
| EC2 Security Group | Application Port, Source는 ALB Security Group | ALB를 통해서만 Application에 접근하게 합니다. |
| EC2 관리 Rule | TCP 22, 관리자 Public IP `/32` | 실습에서 SSH가 필요할 때만 추가합니다. |

운영 환경에서는 SSH Rule 대신 Systems Manager Session Manager를 사용하면 Public SSH 노출과 Key 배포를 줄일 수 있습니다.

## 5. Launch Template 생성

---

Launch Template은 EC2 Instance의 생성 조건을 Version으로 관리합니다. Launch Configuration은 이전 방식이며, 2024년 10월 1일 이후 생성된 AWS Account에서는 새 Launch Configuration을 만들 수 없습니다. 신규 구성은 Launch Template을 사용합니다.

`[EC2] → [시작 템플릿] → [시작 템플릿 생성]`에서 다음 값을 설정합니다.

| 항목 | 설정과 확인 내용 |
| --- | --- |
| 이름과 Version 설명 | 변경 목적을 식별할 수 있게 작성합니다. |
| AMI | 앞에서 생성한 Custom AMI를 선택합니다. |
| Instance Type | AMI Architecture와 호환되는 Type을 선택합니다. |
| Key Pair | SSH가 필요한 실습에서만 지정합니다. |
| Security Group | Auto Scaling용 EC2 Security Group을 지정합니다. |
| Storage | AMI의 Block Device Mapping과 추가 Volume을 확인합니다. |
| IAM Instance Profile | Application이 AWS API를 호출할 때 필요한 최소 권한 Role을 지정합니다. |
| User Data | Boot 시 추가로 실행할 Script가 있을 때 사용합니다. |
| Resource Tag | Instance와 Volume에 전달할 Tag를 지정합니다. |

Auto Scaling Group에서 Subnet을 선택할 예정이라면 Launch Template에 특정 Subnet을 고정하지 않습니다. Group의 Network 설정에서 여러 Availability Zone의 Subnet을 선택해야 분산 배치할 수 있습니다.

Template을 수정할 때 기존 Version을 덮어쓰지 않고 새 Version을 생성합니다. Auto Scaling Group이 어떤 Version을 사용하는지도 함께 확인합니다.

## 6. Auto Scaling Group 생성

---

`[EC2] → [Auto Scaling 그룹] → [Auto Scaling 그룹 생성]`에서 다음 순서로 구성합니다.

### 6.1 Launch Template과 Network

1. 앞에서 만든 Launch Template과 사용할 Version을 선택합니다.

2. EC2 Instance를 배치할 VPC를 선택합니다.

3. 서로 다른 Availability Zone에 있는 Subnet을 두 개 이상 선택합니다.

4. Instance Type 요구 조건과 선택한 Subnet에서 사용할 수 있는 Capacity를 확인합니다.

Launch Template에 AMI 또는 필수 설정이 빠져 있으면 Auto Scaling Group을 만들 수 있어도 Instance 시작이 실패할 수 있습니다. 생성 후 Activity History에서 시작 실패 원인을 확인합니다.

### 6.2 Load Balancer 연결

기존 ALB를 사용한다면 ALB 자체가 아니라 해당 ALB Listener가 전달하는 Target Group을 선택합니다.

- 새 Instance가 생성되면 Auto Scaling Group이 Target Group에 자동으로 등록합니다.

- Instance가 종료되면 Target Group에서도 자동으로 등록 해제합니다.

- `Elastic Load Balancing 상태 확인`을 활성화하면 Target Group의 Health Check 결과를 Instance 교체 판단에 사용할 수 있습니다.

- Health Check Grace Period는 Application Boot와 초기화 시간보다 짧지 않게 설정합니다.

### 6.3 Capacity 설정

| 값 | 실습 예시 | 의미 |
| --- | ---: | --- |
| Minimum Capacity | 2 | 장애나 Scale-in 이후에도 유지할 최소 수입니다. |
| Desired Capacity | 2 | Group이 현재 유지하려는 Instance 수입니다. |
| Maximum Capacity | 4 | Scaling Policy가 늘릴 수 있는 상한입니다. |

실습 예시는 두 Availability Zone에 Instance를 분산하기 위한 값입니다. 실제 값은 장애 시 남아야 하는 Capacity와 비용 한도를 기준으로 정합니다.

### 6.4 Notification과 Tag

필요하면 SNS Topic을 연결해 Instance 시작, 종료와 오류 Event를 알립니다. `Name`, Environment와 Cost 관련 Tag를 Instance에 전파하도록 설정하면 자동 생성된 Resource를 추적하기 쉽습니다.

마지막 검토 화면에서 Launch Template Version, Subnet, Target Group, Health Check와 Capacity 값을 확인한 뒤 Group을 생성합니다.

## 7. 생성 결과 검증

---

### 7.1 Desired Capacity 확인

다음 항목을 Management Console에서 확인합니다.

1. Auto Scaling Group의 Instance Management 화면에 Desired Capacity만큼 Instance가 생성되었는지 확인합니다.

2. Instance가 서로 다른 Availability Zone에 분산되었는지 확인합니다.

3. Target Group에서 각 Instance가 `healthy`인지 확인합니다.

4. ALB DNS Name으로 반복 접속해 Application 응답을 확인합니다.

### 7.2 Instance 교체 확인

Auto Scaling Group이 관리하는 Instance 하나를 종료합니다. Desired Capacity보다 Instance가 줄어들면 Group이 새 Instance를 생성합니다.

```text
Instance 종료
    ↓
Auto Scaling Group이 Capacity 부족 감지
    ↓
Launch Template으로 새 Instance 생성
    ↓
Target Group 등록과 Health Check
    ↓
healthy 상태 이후 Traffic 수신
```

Target Group에서 먼저 수동으로 등록 해제하면 Load Balancer 동작과 Auto Scaling 교체 동작이 섞여 원인을 판단하기 어렵습니다. Instance 교체 검증은 Auto Scaling Group의 Activity History, EC2 Instance 상태와 Target Health를 함께 확인합니다.

### 7.3 CPU 부하를 이용한 Scale-out 확인

CPU 기반 Scaling Policy를 검증할 때는 종료 시점이 있는 부하를 사용합니다. 다음 명령은 Ubuntu Instance의 CPU Core 하나에 최대 5분 동안 부하를 줍니다.

```bash
timeout 300 sh -c 'while :; do :; done'
```

다음 항목을 확인합니다.

- CloudWatch의 CPU Metric이 Policy Threshold를 넘었는지 확인합니다.

- Alarm이 `ALARM` 상태로 전환되었는지 확인합니다.

- Auto Scaling Activity에서 Scale-out이 시작되었는지 확인합니다.

- 새 Instance가 Target Group에서 `healthy`가 되었는지 확인합니다.

한 Core만 사용하는 명령이므로 여러 vCPU를 가진 Instance에서는 전체 평균 CPU가 예상보다 낮을 수 있습니다. 실습이 끝나면 `timeout`이 종료되었는지 확인합니다.

## 8. Scaling 방식

---

| 방식 | 사용 시점 | 동작 |
| --- | --- | --- |
| Target Tracking | CPU나 요청 수를 목표값 근처에 유지할 때 | Metric 목표에 맞춰 Capacity를 자동 조절합니다. |
| Step Scaling | Alarm 크기에 따라 증감 폭을 다르게 할 때 | Threshold 구간별로 정한 수만큼 조절합니다. |
| Scheduled Scaling | 행사나 업무 시간처럼 시점을 알고 있을 때 | 지정한 시간에 Minimum, Maximum 또는 Desired Capacity를 변경합니다. |
| Predictive Scaling | 일별·주별 반복 Pattern이 있을 때 | 과거 Metric을 분석해 필요한 Capacity를 예측하고 미리 Scale-out합니다. |

Predictive Scaling은 Forecast를 만들기 위해 최소 24시간의 Metric Data가 필요하며 최대 14일의 Data를 분석합니다. 처음에는 Forecast Only Mode로 예측 정확도를 확인한 뒤 실제 Scaling에 사용합니다.

## 9. Auto Scaling의 한계와 보완

---

Auto Scaling이 Instance를 시작해 Application을 준비하는 데는 시간이 걸립니다. Traffic 증가 속도가 Instance 준비 속도보다 빠르면 Scale-out이 시작되었더라도 오류나 지연이 발생할 수 있습니다.

- 예상 가능한 Peak 전에는 Scheduled Scaling으로 Capacity를 먼저 늘립니다.

- 반복 Pattern이 있다면 Predictive Scaling과 Dynamic Scaling을 함께 검토합니다.

- Minimum Capacity에 즉시 필요한 여유분을 포함합니다.

- AMI에 공통 Package와 Application을 포함해 Bootstrap 시간을 줄입니다.

- Default Instance Warmup과 Health Check Grace Period를 실제 시작 시간에 맞춥니다.

- CloudFront, Cache와 Queue를 이용해 갑작스러운 부하가 Application Instance로 바로 집중되지 않게 합니다.

## 10. Resource 정리

---

의존 관계가 있는 Resource는 다음 순서로 정리합니다.

1. Auto Scaling Group의 Scale-in Protection을 확인하고 Group을 삭제합니다.

2. 더 이상 사용하지 않는 Launch Template과 Version을 삭제합니다.

3. ALB를 삭제합니다.

4. Target Group을 삭제합니다.

5. 다른 Resource에서 사용하지 않는 Security Group을 삭제합니다.

6. Custom AMI를 Deregister합니다.

7. AMI와 Backup 정책에 필요한지 확인한 뒤 EBS Snapshot을 삭제합니다.

8. 남은 EBS Volume, Elastic IP와 CloudWatch Alarm을 확인합니다.

AMI와 Snapshot을 무조건 함께 삭제하면 복구 지점을 잃을 수 있습니다. 보존 정책과 다른 Launch Template의 참조 여부를 확인한 뒤 삭제합니다.

다음 글인 [EC2 Database 구축과 Amazon RDS](/cloud-07-aws-ec2-database-rds/)에서는 EC2에 MySQL·MongoDB를 직접 설치하는 방식과 Amazon RDS의 관리 범위를 비교합니다.

## 참고 자료

---

- [Amazon EBS Snapshot](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html)

- [Amazon Data Lifecycle Manager](https://docs.aws.amazon.com/ebs/latest/userguide/dlm-elements.html)

- [Auto Scaling용 Launch Template 생성](https://docs.aws.amazon.com/autoscaling/ec2/userguide/create-launch-template.html)

- [Launch Configuration 제한](https://docs.aws.amazon.com/autoscaling/ec2/userguide/launch-configurations.html)

- [Auto Scaling 방식 선택](https://docs.aws.amazon.com/autoscaling/ec2/userguide/scaling-overview.html)

- [Predictive Scaling 동작](https://docs.aws.amazon.com/autoscaling/ec2/userguide/predictive-scaling-policy-overview.html)
