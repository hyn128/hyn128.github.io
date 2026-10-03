---
title: Nginx로 이해하는 ECS Task·Service와 ALB
description: ECS의 Cluster·Capacity·Task Definition·Task·Service 관계를 구분하고 Fargate에서 Nginx를 단독 Task와 ALB 기반 Service로 배포한 뒤 배포 전략과 Auto Scaling을 확인합니다
date: 2026-10-02
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> Amazon ECS는 Container를 어떤 환경에서 어떤 설정으로 몇 개 실행할지 관리합니다. Nginx를 Fargate Task로 먼저 실행해 Task의 수명과 Network를 확인한 뒤, ECS Service와 Application Load Balancer를 연결해 장애 복구·배포·확장이 가능한 구조로 전환합니다.

## 1. ECS 실행 구조

---

ECS Resource는 다음 관계로 동작합니다.

```text
ECS Cluster
├── Capacity Provider
│   └── Container를 실행할 Compute 환경
├── Task Definition
│   └── Container Image, CPU, Memory, Port와 실행 권한 정의
└── ECS Service
    ├── 지정한 수의 Task 유지
    ├── Load Balancer와 연결
    ├── 새 Task Definition Revision 배포
    └── Service Auto Scaling
```

Task Definition은 실행 설정을 담은 Template이며, Task는 그 Template을 이용해 실제로 실행한 단위입니다. Kubernetes의 Pod나 Docker Compose와 목적이 일부 비슷하지만 Resource Model과 동작이 동일하지 않으므로 같은 개념으로 취급하지 않습니다.

| Resource | 역할 | 수명과 변경 방식 |
| --- | --- | --- |
| Cluster | Task와 Service를 논리적으로 묶는 경계 | Capacity Provider와 Service를 포함합니다. |
| Capacity Provider | Task를 실행할 Compute Capacity 연결 | Fargate, Fargate Spot, EC2 Auto Scaling Group 등을 연결합니다. |
| Task Definition | 하나 이상의 Container 실행 설정 | 수정 대신 새 Revision을 등록합니다. |
| Task | Task Definition Revision의 실행 Instance | 종료되면 같은 Task를 다시 시작하지 않습니다. |
| Service | 원하는 Task 수를 유지하는 Controller | 실패한 Task를 교체하고 배포와 Auto Scaling을 수행합니다. |

## 2. Capacity 선택

---

현재 ECS에서 선택할 수 있는 대표적인 Capacity 방식은 다음과 같습니다.

| 방식 | Compute 관리 주체 | 적합한 경우 |
| --- | --- | --- |
| AWS Fargate | AWS | EC2 Instance 운영 없이 Task 단위 CPU·Memory를 지정할 때 사용합니다. |
| Fargate Spot | AWS | 중단을 허용할 수 있는 비용 민감형 Workload에 사용합니다. |
| EC2 Auto Scaling Group Capacity Provider | 사용자 | Instance Type, AMI, Storage와 Host 구성을 직접 제어해야 할 때 사용합니다. |
| Amazon ECS Managed Instances | AWS | EC2 기능을 이용하면서 Provisioning·Scaling·Patch 관리 부담을 줄일 때 사용합니다. |

자체 관리형 EC2 Capacity는 Fargate의 사양을 변경하는 방식이 아닙니다. 사용자가 만든 EC2 Auto Scaling Group을 Capacity Provider로 연결하고 Instance Capacity를 직접 관리하는 방식입니다.

이 글에서는 Host 관리가 필요 없는 Fargate를 사용합니다.

## 3. 배포 전 준비

---

다음 Resource가 필요합니다.

| Resource | 예제 값 | 목적 |
| --- | --- | --- |
| ECS Cluster | `web-cluster` | Task와 Service의 논리적 경계 |
| VPC | `<VPC_ID>` | ALB와 Task가 통신할 Network |
| Public Subnet | 두 개 이상의 가용 영역 | Internet-facing ALB 배치 |
| Private Subnet | 두 개 이상의 가용 영역 | Service Task 배치 |
| ALB Security Group | `alb-sg` | Client의 HTTP·HTTPS 요청 허용 |
| Task Security Group | `nginx-task-sg` | ALB에서 오는 Nginx Traffic만 허용 |
| Task Execution Role | `ecsTaskExecutionRole` | ECR Image Pull과 CloudWatch Logs 전송 |

학습 환경에서 Public Subnet에 Task를 두고 Public IP를 부여할 수 있습니다. 운영 구조에서는 ALB만 Public Subnet에 두고 Task는 Private Subnet에 배치하는 방식이 일반적입니다. Private Subnet의 Task가 Public ECR나 외부 Package Repository에 접근해야 한다면 NAT Gateway 또는 필요한 VPC Endpoint를 구성합니다.

## 4. Cluster 생성

---

AWS Console의 **Amazon ECS → Clusters → Create cluster**에서 Cluster 이름을 `web-cluster`로 지정합니다. Fargate Task만 사용할 때 EC2 Instance를 미리 생성할 필요는 없습니다.

AWS CLI에서는 다음과 같이 생성할 수 있습니다.

```bash
aws ecs create-cluster \
  --cluster-name web-cluster
```

Cluster가 `ACTIVE` 상태인지 확인합니다.

```bash
aws ecs describe-clusters \
  --clusters web-cluster \
  --query 'clusters[0].{name:clusterName,status:status}'
```

## 5. Nginx Task Definition 등록

---

Task Definition은 Nginx Container의 Image, Port, Resource와 Log Driver를 정의합니다. 다음 내용을 `nginx-task-definition.json`으로 저장합니다.

```json
{
  "family": "nginx-web",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "public.ecr.aws/docker/library/nginx:stable-alpine",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/nginx-web",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

주요 속성의 역할은 다음과 같습니다.

| 속성 | 역할 |
| --- | --- |
| `family` | Task Definition Revision을 묶는 이름입니다. |
| `requiresCompatibilities` | Fargate에서 실행할 수 있는 정의인지 검증합니다. |
| `networkMode: awsvpc` | 각 Task에 Elastic Network Interface와 Private IP를 할당합니다. |
| `cpu`, `memory` | Task 전체에 예약할 Resource 조합입니다. |
| `executionRoleArn` | ECS Agent가 Image와 Log Service에 접근할 때 사용합니다. |
| `essential` | 이 Container가 중지되면 Task도 중지하도록 지정합니다. |

Log Group을 먼저 만들고 Task Definition을 등록합니다.

```bash
aws logs create-log-group \
  --log-group-name /ecs/nginx-web

aws ecs register-task-definition \
  --cli-input-json file://nginx-task-definition.json
```

등록한 Revision을 확인합니다.

```bash
aws ecs describe-task-definition \
  --task-definition nginx-web \
  --query 'taskDefinition.{family:family,revision:revision,status:status}'
```

## 6. 단독 Task 실행

---

단독 Task는 Batch 작업이나 일회성 검증에 적합합니다. 종료된 Task를 ECS Service가 자동으로 교체하지 않는다는 점을 확인하기 위해 먼저 이 방식으로 Nginx를 실행합니다.

Security Group의 Inbound Rule은 실습 Client의 Public IP에서 오는 TCP 80만 허용합니다. `0.0.0.0/0`으로 관리 Port까지 함께 개방하지 않습니다.

```text
Type: HTTP
Protocol: TCP
Port: 80
Source: <CLIENT_PUBLIC_IP>/32
```

Public Subnet에서 다음 명령을 실행합니다. Placeholder는 실제 Resource ID로 교체합니다.

```bash
aws ecs run-task \
  --cluster web-cluster \
  --launch-type FARGATE \
  --task-definition nginx-web \
  --network-configuration 'awsvpcConfiguration={subnets=[<PUBLIC_SUBNET_ID>],securityGroups=[<TASK_SECURITY_GROUP_ID>],assignPublicIp=ENABLED}'
```

Task가 `RUNNING`인지 확인합니다.

```bash
aws ecs list-tasks \
  --cluster web-cluster \
  --family nginx-web \
  --desired-status RUNNING
```

Console에서 Task의 Public IP를 확인한 뒤 Nginx 응답을 검증합니다.

```bash
curl -I http://<TASK_PUBLIC_IP>/
```

`HTTP/1.1 200 OK`가 반환되면 Network와 Container Port가 연결된 상태입니다. Task를 중지하면 이 Public IP는 더 이상 사용할 수 없으며 새 Task에도 같은 주소가 보장되지 않습니다.

## 7. ALB 기반 Service 구성

---

지속적으로 제공할 Application은 단독 Task 대신 Service로 실행합니다.

```text
Internet Client
  ↓ TCP 80 또는 443
Application Load Balancer
  ↓ ALB Listener와 Rule
IP Target Group
  ↓ TCP 80
Fargate Task 1 · Fargate Task 2
```

Fargate의 `awsvpc` Network Mode에서는 각 Task가 IP를 가지므로 Target Group의 Target Type을 `ip`로 생성합니다.

Security Group은 호출 방향에 맞춰 분리합니다.

| Security Group | Inbound Rule | Source |
| --- | --- | --- |
| `alb-sg` | TCP 80·443 | 필요한 Client 대역 또는 Internet |
| `nginx-task-sg` | TCP 80 | `alb-sg`의 Security Group ID |

Task Security Group의 Source를 ALB Security Group으로 지정하면 ALB를 통과하지 않은 직접 요청을 차단할 수 있습니다.

Target Group의 Health Check는 다음과 같이 설정합니다.

| 항목 | 값 |
| --- | --- |
| Protocol | HTTP |
| Path | `/` |
| Port | Traffic port |
| Success code | `200` |

ECS Console의 **Cluster → Services → Create**에서 다음 값을 지정합니다.

1. Compute option은 `Launch type`, Launch type은 `FARGATE`를 선택합니다.

2. Task Definition Family는 `nginx-web`, Revision은 최신 Revision을 선택합니다.

3. Service 이름은 `nginx-service`, Desired tasks는 `2`로 지정합니다.

4. 두 개 이상의 Private Subnet과 `nginx-task-sg`를 선택합니다.

5. Application Load Balancer, Listener, `ip` Target Group과 Container Port 80을 연결합니다.

Service가 안정화됐는지 확인합니다.

```bash
aws ecs describe-services \
  --cluster web-cluster \
  --services nginx-service \
  --query 'services[0].{desired:desiredCount,running:runningCount,pending:pendingCount,events:events[0:3]}'
```

Target Health와 ALB 응답도 확인합니다.

```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN>

curl -I http://<ALB_DNS_NAME>/
```

## 8. Task Definition Revision과 Service 배포

---

Task Definition은 기존 Revision을 덮어쓰지 않습니다. Image나 설정을 바꾼 JSON을 등록하면 새 Revision이 생성됩니다.

```bash
aws ecs register-task-definition \
  --cli-input-json file://nginx-task-definition.json
```

Service가 새 Revision을 사용하도록 갱신합니다.

```bash
aws ecs update-service \
  --cluster web-cluster \
  --service nginx-service \
  --task-definition nginx-web:<NEW_REVISION>
```

진행 중인 Deployment를 확인합니다.

```bash
aws ecs describe-services \
  --cluster web-cluster \
  --services nginx-service \
  --query 'services[0].deployments[*].{status:status,taskDefinition:taskDefinition,desired:desiredCount,running:runningCount,failed:failedTasks}'
```

현재 ECS Service의 Deployment Strategy는 다음과 같습니다.

| Strategy | 동작 | 용도 |
| --- | --- | --- |
| Rolling | 기존 Task를 새 Task로 점진적으로 교체합니다. | 기본적인 무중단 배포 |
| Blue/Green | 새 Revision의 Green 환경을 검증한 뒤 Traffic을 전환합니다. | 명확한 전환과 빠른 Rollback |
| Linear | 일정 비율씩 Traffic을 이동합니다. | 단계적 위험 제어 |
| Canary | 일부 Traffic으로 먼저 검증한 뒤 나머지를 전환합니다. | 실제 Traffic 기반 사전 검증 |

Blue/Green은 새 Task Definition Revision으로 Green Task Set을 만들고 Health Check를 통과한 뒤 Traffic을 전환합니다. 전환 중에는 Blue와 Green을 함께 실행할 Capacity가 필요합니다. 기존 CodeDeploy 기반 Blue/Green도 지원되지만, 새 Service 구성에서는 ECS Deployment Controller가 제공하는 Native Blue/Green·Linear·Canary 기능을 우선 검토합니다.

## 9. Service Auto Scaling

---

Service Auto Scaling은 Metric에 따라 Desired Count를 변경합니다. Task 내부의 Process 수를 바꾸는 기능이 아닙니다.

| Metric | 의미 | 주의 사항 |
| --- | --- | --- |
| `ECSServiceAverageCPUUtilization` | Service Task의 평균 CPU 사용률 | CPU가 병목인 Workload에 사용합니다. |
| `ECSServiceAverageMemoryUtilization` | Service Task의 평균 Memory 사용률 | Memory 사용량이 부하와 비례하는지 확인합니다. |
| `ALBRequestCountPerTarget` | Target 하나당 요청 수 | ECS Blue/Green Target Tracking에는 사용할 수 없습니다. |
| Custom Metric | Application Queue나 처리량 | CloudWatch에 Metric을 별도로 발행해야 합니다. |

Target Tracking Policy에서는 최소·최대 Task 수와 Target 값을 지정합니다.

```text
Minimum tasks: 2
Maximum tasks: 6
Target metric: Average CPU utilization 60%
Scale-out cooldown: 60 seconds
Scale-in cooldown: 180 seconds
```

Scale-out Cooldown은 확장 직후 추가 확장 판단에 새 Capacity가 반영될 시간을 줍니다. Scale-in Cooldown은 축소 직후 너무 빠르게 다시 축소하지 않도록 합니다. 배포 중에는 Scale-out이 가능하지만 Service Auto Scaling의 Scale-in은 일시 중단됩니다.

여러 Target Tracking Policy를 함께 사용하면 어느 하나라도 확장을 요구할 때 Scale-out하며, 모든 Policy가 축소에 동의할 때 Scale-in합니다.

## 10. 장애와 배포 상태 점검

---

Service 상태만 보지 않고 Event, 중지된 Task 이유, Target Health와 Log를 함께 확인합니다.

```bash
aws ecs describe-services \
  --cluster web-cluster \
  --services nginx-service \
  --query 'services[0].events[0:10]'

aws ecs list-tasks \
  --cluster web-cluster \
  --service-name nginx-service \
  --desired-status STOPPED

aws logs tail /ecs/nginx-web \
  --since 10m
```

대표적인 실패 원인은 다음과 같습니다.

| 증상 | 확인할 항목 |
| --- | --- |
| Task가 `PENDING`에 머묾 | Subnet의 IP 여유, Fargate CPU·Memory 조합, Service Quota |
| Image Pull 실패 | Execution Role, Image URI, NAT Gateway 또는 ECR VPC Endpoint |
| Target이 `unhealthy` | Health Check Path, Container Port, Task Security Group |
| ALB 응답 없음 | Listener Rule, ALB Security Group, Target Group 상태 |
| 새 Revision이 Rollback됨 | Container Log, Health Check Grace Period, Deployment Circuit Breaker |

## 11. Resource 정리

---

비용이 계속 발생하지 않도록 실습 Resource의 의존 순서에 맞춰 정리합니다.

1. ECS Service의 Desired Count를 0으로 변경하거나 Service를 삭제합니다.

2. 단독 실행한 Task가 남아 있으면 중지합니다.

3. ALB Listener, Target Group과 ALB를 삭제합니다.

4. 사용하지 않는 Log Group, Security Group과 ECS Cluster를 삭제합니다.

5. NAT Gateway를 실습용으로 만들었다면 Elastic IP와 함께 삭제 여부를 확인합니다.

> **최종 정리**
> - Task Definition은 실행 Template이고 Task는 그 Revision으로 생성된 실행 단위입니다.
>
> - 단독 Task는 종료돼도 자동 교체되지 않지만 Service는 Desired Count를 유지합니다.
>
> - Fargate Task는 `awsvpc` Network Mode에서 IP를 가지므로 ALB Target Group은 `ip` Type을 사용합니다.
>
> - 새 Application Version은 새 Task Definition Revision으로 등록하고 Service Deployment로 교체합니다.
>
> - Service Auto Scaling은 Metric, 최소·최대 Task 수와 Cooldown을 함께 설계합니다.

## 참고 자료

---

- [Amazon ECS Capacity option 비교](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/capacity-launch-type-comparison.html)

- [Amazon ECS Task Definition](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)

- [Amazon ECS Service Deployment Strategy](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_service-options.html)

- [Amazon ECS Blue/Green Deployment](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/blue-green-deployment-implementation.html)

- [Amazon ECS Service Auto Scaling](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html)

다음 글에서는 [Django Image를 ECR와 ECS에 배포하는 GitHub Actions Pipeline]({% post_url 2026-10-02-cloud-15-django-ecr-ecs-github-actions %})을 구성합니다.
