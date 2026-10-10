---
title: AWS Hybrid Cloud와 VPC Network 설계
description: On-Premise와 AWS를 연결하는 Hybrid Cloud Pattern을 구분하고 Direct Connect, Site-to-Site VPN, Outposts와 VPC Network의 Traffic 흐름을 정리합니다
date: 2026-10-07
updated_at: 2026-10-08
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> Hybrid Cloud는 On-Premise 환경과 Public Cloud를 하나의 운영 범위로 연결하는 구조입니다. Cloud Bursting, Disaster Recovery, Data Tiering과 Edge Computing Pattern을 목적에 맞게 선택하고 AWS Direct Connect, Site-to-Site VPN 또는 AWS Outposts로 연결합니다. VPC 안에서는 Subnet, Route Table, Internet Gateway, NAT Gateway, Security Group과 Network ACL이 Traffic 경로와 허용 범위를 나누어 담당합니다.

## 1. Hybrid Cloud

---

Hybrid Cloud는 기존 Data Center의 System과 Public Cloud Resource를 Network, Identity와 운영 체계로 연결한 환경입니다. 단순히 두 환경을 보유하는 것만으로는 부족하며 Workload와 Data가 정해진 정책에 따라 이동하거나 서로 통신할 수 있어야 합니다.

도입을 검토하는 대표 이유는 다음과 같습니다.

- 기존 Data Center와 Legacy System 투자를 유지해야 하는 경우

- Data 주권, 보안 또는 규제 요구로 일부 Data를 On-Premise에 보관해야 하는 경우

- 현장에서 매우 낮은 지연 시간이 필요한 경우

- ERP, Groupware와 Directory Service를 Cloud Workload와 연결해야 하는 경우

- 한 번에 전체 System을 이전하지 않고 단계적으로 Migration해야 하는 경우

## 2. 핵심 구축 Pattern

---

| Pattern | 주요 목적 | Data 동기화 | 대표 Workload |
| --- | --- | --- | --- |
| Cloud Bursting | 평상시 Capacity를 넘는 수요를 Cloud로 확장 | Stateless 처리 또는 빠른 상태 공유 필요 | Promotion Traffic, Rendering과 Batch |
| Disaster Recovery | 주 Site 장애 시 Cloud에서 업무 복구 | Recovery 목표에 맞춘 지속 또는 주기적 복제 | Database, ERP와 Groupware |
| Data Tiering | 고가 On-Premise Storage 사용량 절감 | 주기적 Lifecycle과 Batch 이동 | Log, Archive와 Compliance 보관 |
| Distributed Edge Computing | 현장 저지연 처리 후 결과만 중앙 전송 | Local Cache와 요약 Data 전송 | Smart Factory, Video 분석과 현장 제어 |

### 2.1 Cloud Bursting

기본 Traffic은 On-Premise에서 처리하고 Peak 수요만 Cloud Resource로 확장합니다. Application 상태를 특정 Server에 묶지 않거나 상태를 공유할 수 있어야 Traffic을 두 환경에 분산하기 쉽습니다.

### 2.2 Disaster Recovery

Primary Workload는 On-Premise에 두고 AWS를 Backup 또는 Standby Site로 사용합니다. 단순 Backup 보관과 실제 Service 전환은 다르므로 Recovery Point Objective와 Recovery Time Objective에 맞춰 Replication, Failover와 정기 복구 Test를 설계합니다.

### 2.3 Data Tiering

빈번하게 사용하는 Transaction Data는 On-Premise SSD에 두고 오래된 Data는 S3 같은 Object Storage로 이동합니다. 이동 주기, 암호화, Retention과 복원 시간을 함께 정의해야 합니다.

### 2.4 Distributed Edge Computing

공장이나 현장 Node에서 저지연 판단을 수행하고 필요한 결과만 Cloud로 전송합니다. WAN 연결이 끊겨도 Local 제어가 계속 동작하도록 Offline 동작과 재동기화 절차를 설계합니다.

## 3. 구축에 필요한 공통 요소

---

| 요소 | 역할 | 설계 관점 |
| --- | --- | --- |
| Hybrid Network | On-Premise와 VPC 경로 연결 | Bandwidth, 지연 시간, 암호화와 이중화 |
| 통합 실행 Platform | 환경 간 Application 배포 방식 통일 | Kubernetes, EKS와 OpenShift 등의 Image·Manifest 호환성 |
| Identity와 Security | 환경 전체의 인증·인가와 Data 보호 | 최소 권한, 전송·저장 암호화와 감사 Log |

Direct Connect가 모든 연결에 고정된 속도를 보장하는 것은 아닙니다. Port Capacity, Location, Virtual Interface와 회선 사업자 구간을 포함한 전체 경로를 기준으로 Bandwidth와 이중화를 설계합니다.

## 4. AWS Hybrid 연결 방식

---

| 방식 | Network 경로 | 주요 용도 | 주의 사항 |
| --- | --- | --- | --- |
| AWS Site-to-Site VPN | Public Internet 위 IPsec Tunnel | 빠른 초기 연결, 작은 Traffic, Backup 경로 | Internet 품질의 영향을 받음 |
| AWS Direct Connect | 전용 Network 연결 | 일관된 Network 경험, 지속적 대용량 Traffic | 회선 구축 기간과 이중화 필요 |
| AWS Outposts | On-Premise에 AWS 관리형 Infrastructure 배치 | Local 저지연 처리와 AWS API 일관성 | Home Region 연결과 현장 시설 요건 필요 |

Site-to-Site VPN은 Customer Gateway와 AWS 측 Virtual Private Gateway 또는 Transit Gateway 사이에 IPsec Tunnel을 구성합니다. Direct Connect는 Private Virtual Interface 또는 Transit Virtual Interface 등을 통해 AWS Network에 연결합니다. 여러 VPC와 여러 Site를 중앙에서 연결할 때는 Transit Gateway를 Hub로 사용할 수 있습니다.

Direct Connect는 기본적으로 Application Traffic 암호화를 자동 제공하는 Service가 아닙니다. 암호화가 필요하면 지원 조건을 확인해 MACsec 또는 Direct Connect 위 Site-to-Site VPN 같은 구성을 검토합니다.

Outposts는 독립된 AWS Region이 아닙니다. VPC를 On-Premise Outpost로 확장하며 관리 Traffic과 Region Service 접근을 위해 Home Region과 지속적인 Service Link가 필요합니다.

## 5. Region, Availability Zone과 VPC

---

### 5.1 Region과 Availability Zone

Region은 지리적으로 분리된 AWS 운영 영역이고, 한 Region은 여러 Availability Zone으로 구성됩니다. Availability Zone은 독립된 전력, 냉각과 물리 보안을 갖춘 하나 이상의 Data Center로 구성되며 Region 내부의 다른 Zone과 전용 Network로 연결됩니다.

Region과 Availability Zone 수는 계속 변하므로 고정된 숫자를 설계 기준으로 사용하지 않습니다. 배포 시점의 [AWS Global Infrastructure](https://aws.amazon.com/about-aws/global-infrastructure/)에서 대상 Region의 Zone 구성을 확인합니다.

### 5.2 Amazon VPC

Amazon VPC는 AWS Account 안에 만드는 논리적으로 격리된 Virtual Network입니다. VPC에 IPv4 CIDR을 지정하고 Subnet, Route와 Security Control을 조합해 Resource의 통신 범위를 정의합니다.

```text
AWS Region
└── VPC 10.20.0.0/16
    ├── AZ-A
    │   ├── Public Subnet
    │   └── Private Subnet
    └── AZ-B
        ├── Public Subnet
        └── Private Subnet
```

On-Premise Network와 VPC CIDR가 겹치면 Route를 명확히 결정할 수 없습니다. Hybrid 연결 전에는 기존 Data Center, 지사, VPN Client와 연결 대상 VPC의 CIDR 중복을 먼저 확인합니다.

## 6. Public Subnet과 Private Subnet

---

Subnet의 Public·Private 구분은 이름이 아니라 연결된 Route Table로 결정됩니다.

| 구분 | Route | 대표 Resource |
| --- | --- | --- |
| Public Subnet | `0.0.0.0/0 → Internet Gateway` | Internet-facing Load Balancer, NAT Gateway |
| Private Subnet | Internet Gateway 직접 Route 없음 | Application Server, Cache와 Database |

Public Subnet에 배치했다는 이유만으로 Instance가 Internet과 통신하는 것은 아닙니다. IPv4 통신에는 Public IPv4 또는 Elastic IP와 Security Group 허용 규칙도 필요합니다.

Private Subnet의 Resource가 Package Update나 외부 API 호출을 위해 IPv4 Internet Outbound 통신을 해야 한다면 Public NAT Gateway를 사용합니다.

```text
Private Instance
  ↓ Private Subnet Route Table
NAT Gateway in Public Subnet
  ↓ Public Subnet Route Table
Internet Gateway
  ↓
Internet
```

Public NAT Gateway는 Public Subnet에 만들고 Elastic IP를 연결합니다. Availability Zone 장애 격리와 불필요한 Cross-AZ Traffic을 줄이려면 각 Zone에 NAT Gateway를 두고 같은 Zone의 Private Subnet이 사용하도록 Route를 구성합니다.

## 7. Internet Gateway와 Route Table

---

Internet Gateway는 VPC에 연결하는 수평 확장형 Managed Component입니다. Public Subnet의 Route Table에서 Internet Traffic의 Target으로 지정합니다.

IPv4에서 Internet Gateway는 Instance의 Private IPv4와 연결된 Public IPv4 또는 Elastic IP 사이의 1:1 NAT를 논리적으로 수행합니다. Internet Gateway를 VPC에 연결하는 것만으로 Traffic이 허용되지는 않으며 Route, Public Address, Security Group과 Network ACL 조건을 모두 충족해야 합니다.

Route Table은 Destination CIDR와 Target의 집합입니다.

| Destination | Target | 의미 |
| --- | --- | --- |
| `10.20.0.0/16` | `local` | VPC 내부 통신 |
| `0.0.0.0/0` | Internet Gateway | Public IPv4 Internet 경로 |
| `0.0.0.0/0` | NAT Gateway | Private Subnet의 IPv4 Outbound 경로 |
| `<ON_PREMISE_CIDR>` | Virtual Private Gateway 또는 Transit Gateway | On-Premise 경로 |

Route는 Traffic의 이동 경로를 결정하고 Security Rule은 해당 Traffic의 허용 여부를 결정합니다. Route가 존재해도 Security Group이나 Network ACL이 차단하면 통신할 수 없습니다.

## 8. Security Group과 Network ACL

---

| 구분 | Security Group | Network ACL |
| --- | --- | --- |
| 적용 위치 | Network Interface와 연결된 Resource | Subnet 경계 |
| 상태 추적 | Stateful | Stateless |
| 규칙 | Allow만 사용 | Allow와 Deny 사용 |
| 평가 | 모든 규칙을 종합 | 낮은 Rule Number부터 순서대로 적용 |
| 반환 Traffic | 허용된 요청의 응답 자동 허용 | Inbound와 Outbound를 각각 허용해야 함 |

Security Group은 Resource 역할을 기준으로 최소 권한을 부여합니다. 예를 들어 Application Security Group의 HTTPS Traffic만 Database Security Group의 Database Port로 허용할 수 있습니다.

Network ACL은 Subnet 단위의 추가 방어선입니다. Stateless이므로 요청 Port뿐 아니라 응답에 사용하는 Ephemeral Port와 반대 방향 규칙도 함께 확인해야 합니다.

## 9. Hybrid End-to-End Traffic 흐름

---

On-Premise Client가 Private VPC Application에 접근하는 흐름은 다음과 같습니다.

```text
On-Premise Client
  ↓ On-Premise Route
Customer Gateway
  ↓ Site-to-Site VPN 또는 Direct Connect
Virtual Private Gateway / Transit Gateway
  ↓ VPC Route Table
Subnet Network ACL
  ↓
Resource Security Group
  ↓
Application Network Interface
```

각 단계의 확인 항목은 다음과 같습니다.

1. On-Premise Router가 VPC CIDR를 AWS 연결로 보내는지 확인합니다.

2. VPN Tunnel 또는 Direct Connect Virtual Interface 상태와 경로 교환을 확인합니다.

3. AWS Gateway와 VPC Route Table에 On-Premise CIDR 반환 경로가 있는지 확인합니다.

4. Network ACL의 Inbound·Outbound 규칙과 Ephemeral Port를 확인합니다.

5. Security Group이 필요한 Protocol과 Port를 On-Premise CIDR 또는 Source Security Group에 허용하는지 확인합니다.

6. Application이 실제 Network Interface와 Port에서 Listen하는지 확인합니다.

한 방향 Route만 구성하면 요청은 도착해도 응답이 돌아가지 못합니다. Hybrid Network는 항상 왕복 경로를 함께 점검합니다.

## 10. 사례에 적용하기

---

### 10.1 금융 System

민감한 원장과 계정계는 규제와 기존 System 요건에 따라 On-Premise에 유지하고, Mobile Channel과 분석 Workload는 AWS에 배치할 수 있습니다. 두 환경은 Direct Connect 또는 VPN으로 연결하고 Identity, 암호화, 감사 Log와 장애 시 우회 경로를 함께 설계합니다.

### 10.2 Smart Factory

Robot Vision과 품질 제어처럼 수 밀리초 단위 판단이 필요한 Workload는 현장 Edge에서 실행합니다. Cloud에는 비식별화한 결과와 학습 Data를 전송하고, Network 단절 중에는 Local 제어를 유지한 뒤 연결 복구 후 재동기화합니다.

이 사례의 성과는 Network 연결 방식만으로 결정되지 않습니다. Application 구조, Data 품질, 운영 자동화와 복구 Test 결과를 별도로 측정해야 합니다.

## 11. 설계 점검 목록

---

- On-Premise, 지사, Client VPN과 VPC CIDR가 겹치지 않는가

- 업무별 Latency, Bandwidth와 암호화 요구를 정의했는가

- Direct Connect 또는 VPN 장애 시 보조 경로와 Failover를 검증했는가

- 여러 VPC 연결에 Virtual Private Gateway와 Transit Gateway 중 적절한 종료 지점을 선택했는가

- Public·Private Subnet Route와 NAT Gateway의 Availability Zone 배치가 일치하는가

- Security Group과 Network ACL의 왕복 Traffic 규칙이 맞는가

- Data Replication 주기와 Recovery 목표를 실제 Test로 검증했는가

- Network, NAT Gateway, Data Transfer와 Direct Connect 비용을 함께 산정했는가

> **최종 정리**
> - Hybrid Cloud는 Workload와 Data가 정책에 따라 On-Premise와 AWS 사이에서 통신하는 운영 구조입니다.
>
> - Site-to-Site VPN은 빠른 도입과 Backup 경로에, Direct Connect는 지속적이고 일관된 연결에 사용합니다.
>
> - Outposts는 AWS Infrastructure를 On-Premise로 확장하지만 Home Region과 Service Link가 필요합니다.
>
> - Public Subnet은 Internet Gateway Route로 결정되며 IPv4 Internet 통신에는 Public Address가 추가로 필요합니다.
>
> - Route Table은 경로를, Network ACL과 Security Group은 Traffic 허용 범위를 결정합니다.

## 참고 자료

---

- [AWS Hybrid Connectivity Service](https://docs.aws.amazon.com/whitepapers/latest/hybrid-connectivity/aws-hybrid-connectivity-services.html)

- [AWS Site-to-Site VPN과 Direct Connect](https://docs.aws.amazon.com/prescriptive-guidance/latest/designing-control-tower-landing-zone/networking.html)

- [AWS Outposts Network 동작](https://docs.aws.amazon.com/outposts/latest/network-userguide/how-outposts-works.html)

- [Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)

- [NAT Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)

- [Security Group과 Network ACL](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Security.html)
