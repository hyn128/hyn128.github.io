---
title: 2-AZ VPC Public·Private Network 구축
description: 두 Availability Zone에 Public·Private Subnet을 나누고 Internet Gateway, NAT Gateway, Route Table과 Security Group을 구성해 접속 경로를 검증합니다
date: 2026-10-08
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> 하나의 Amazon VPC를 두 Availability Zone으로 나누고 각 Zone에 Public Subnet과 Private Subnet을 구성합니다. Public Subnet은 Internet Gateway를 통해 외부 Traffic을 받고, Private Subnet은 같은 Zone의 NAT Gateway를 통해 Outbound Internet에 접근합니다. Route Table과 Security Group을 역할별로 분리하고 Session Manager 또는 제한된 Bastion 경로로 Private EC2 접속을 검증합니다.

## 1. 목표 Architecture

---

이 실습에서 만드는 Network는 다음과 같습니다.

```text
VPC 10.20.0.0/16
├── AZ-A
│   ├── Public Subnet A  10.20.1.0/24
│   │   ├── Internet-facing Web 또는 Bastion
│   │   └── NAT Gateway A
│   └── Private Subnet A 10.20.11.0/24
│       └── Application EC2
└── AZ-B
    ├── Public Subnet B  10.20.2.0/24
    │   ├── Internet-facing Web
    │   └── NAT Gateway B
    └── Private Subnet B 10.20.12.0/24
        └── Application EC2
```

| 표기 | 실행 위치 | 작업 |
| --- | --- | --- |
| `[AWS-CONSOLE]` | AWS Management Console | VPC, Subnet, Gateway, Route와 Security Group 생성 |
| `[LOCAL]` | 관리자 PC | AWS Session Manager 또는 제한된 SSH 접속 |
| `[PUBLIC-EC2]` | Public Subnet의 EC2 | Public 접속과 Private EC2 경로 검증 |
| `[PRIVATE-EC2]` | Private Subnet의 EC2 | NAT Gateway를 통한 Outbound 검증 |

> **CIDR**
> Classless Inter-Domain Routing의 약자입니다. `10.20.0.0/16`처럼 Network 주소와 Prefix 길이로 IP 범위를 표현합니다.

## 2. Resource와 CIDR 계획

---

| Resource | Name Tag | CIDR 또는 연결 대상 |
| --- | --- | --- |
| VPC | `lab-vpc` | `10.20.0.0/16` |
| Public Subnet A | `public-subnet-a` | `10.20.1.0/24` |
| Public Subnet B | `public-subnet-b` | `10.20.2.0/24` |
| Private Subnet A | `private-subnet-a` | `10.20.11.0/24` |
| Private Subnet B | `private-subnet-b` | `10.20.12.0/24` |
| Internet Gateway | `lab-igw` | `lab-vpc`에 Attach |
| NAT Gateway A | `nat-gateway-a` | Public Subnet A와 Elastic IP |
| NAT Gateway B | `nat-gateway-b` | Public Subnet B와 Elastic IP |

VPC CIDR는 On-Premise, Client VPN과 다른 VPC의 CIDR와 겹치지 않아야 합니다. 이후 VPN이나 VPC Peering을 연결할 계획이면 생성 전에 전체 주소 계획을 확인합니다.

## 3. VPC와 Subnet 생성

---

### 3.1 VPC 생성

`[AWS-CONSOLE]`의 VPC 화면에서 `Create VPC`를 누르고 `VPC only`를 선택합니다.

| 항목 | 값 |
| --- | --- |
| Name tag | `lab-vpc` |
| IPv4 CIDR | `10.20.0.0/16` |

DNS 기반 Service Discovery를 사용할 수 있도록 `DNS resolution`과 `DNS hostnames`가 활성화됐는지 확인합니다.

### 3.2 네 Subnet 생성

`lab-vpc`를 선택하고 두 Availability Zone에 네 Subnet을 만듭니다.

| Subnet | Availability Zone | IPv4 CIDR |
| --- | --- | --- |
| `public-subnet-a` | `<AZ_A>` | `10.20.1.0/24` |
| `public-subnet-b` | `<AZ_B>` | `10.20.2.0/24` |
| `private-subnet-a` | `<AZ_A>` | `10.20.11.0/24` |
| `private-subnet-b` | `<AZ_B>` | `10.20.12.0/24` |

Public Subnet 두 개는 `Edit subnet settings`에서 `Enable auto-assign public IPv4 address`를 필요한 경우 활성화합니다. 이 설정만으로 Public Subnet이 되는 것은 아닙니다. Internet Gateway로 향하는 Route가 있어야 합니다.

Private Subnet에서는 Public IPv4 자동 할당을 비활성화합니다.

## 4. Internet Gateway와 Public Route

---

`[AWS-CONSOLE]`에서 `lab-igw` Internet Gateway를 생성해 `lab-vpc`에 Attach합니다.

Public Route Table `public-rt`를 만들고 다음 Route를 확인하거나 추가합니다.

| Destination | Target |
| --- | --- |
| `10.20.0.0/16` | `local` |
| `0.0.0.0/0` | `lab-igw` |

`Subnet associations`에서 `public-subnet-a`, `public-subnet-b`를 연결합니다.

Internet Gateway의 Stateful 여부를 Security Group처럼 해석하지 않습니다. Internet Gateway는 Route Target과 Public IPv4 NAT 기능을 제공하고, Stateful Traffic 허용은 Security Group이 담당합니다.

## 5. NAT Gateway와 Private Route

---

NAT Gateway는 반드시 Public Subnet에 생성합니다. Private Subnet의 Default Route가 NAT Gateway를 가리키도록 구성합니다.

| NAT Gateway | Subnet | Elastic IP | 사용하는 Private Subnet |
| --- | --- | --- | --- |
| `nat-gateway-a` | `public-subnet-a` | 새 Elastic IP | `private-subnet-a` |
| `nat-gateway-b` | `public-subnet-b` | 새 Elastic IP | `private-subnet-b` |

각 NAT Gateway가 `Available` 상태가 된 뒤 Zone별 Private Route Table을 만듭니다.

`private-rt-a`의 Route는 다음과 같습니다.

| Destination | Target |
| --- | --- |
| `10.20.0.0/16` | `local` |
| `0.0.0.0/0` | `nat-gateway-a` |

`private-subnet-a`를 `private-rt-a`에 연결합니다. `private-rt-b`도 같은 방식으로 만들되 Default Route의 Target은 `nat-gateway-b`로 지정하고 `private-subnet-b`를 연결합니다.

실습 비용을 줄이려고 NAT Gateway 하나만 만들 수 있지만 다른 Zone의 Private Subnet이 해당 Gateway를 사용하면 Zone 장애에 취약하고 Cross-AZ Data 처리 비용이 생길 수 있습니다. 운영 환경에서는 Zone별 NAT Gateway와 Route를 사용합니다.

## 6. Security Group 설계

---

Source를 VPC 전체 CIDR보다 역할별 Security Group으로 지정합니다.

### 6.1 Public Web Security Group

`web-sg`의 Inbound Rule은 다음과 같습니다.

| Protocol | Port | Source | 목적 |
| --- | --- | --- | --- |
| TCP | 80 | `0.0.0.0/0` | HTTP Service 공개 |
| TCP | 443 | `0.0.0.0/0` | HTTPS Service 공개 |

Web Server가 아니라 Bastion 용도로만 사용하는 EC2에는 HTTP와 HTTPS를 열지 않습니다.

### 6.2 Bastion Security Group

`bastion-sg`에는 관리자 주소만 SSH를 허용합니다.

| Protocol | Port | Source |
| --- | --- | --- |
| TCP | 22 | `<ADMIN_PUBLIC_IP>/32` |

관리자 주소가 바뀐다는 이유로 `0.0.0.0/0`에 SSH를 공개하지 않습니다. 고정 접속 주소가 없다면 Systems Manager Session Manager를 우선 사용합니다.

### 6.3 Private Application Security Group

`app-sg`는 실제 Source 역할만 허용합니다.

| Protocol | Port | Source | 목적 |
| --- | --- | --- | --- |
| TCP | 22 | `bastion-sg` | Bastion을 사용하는 경우 관리 접속 |
| TCP | `<APP_PORT>` | `web-sg` | Web Tier에서 Application 호출 |
| ICMP | 필요한 Type만 | `bastion-sg` | 선택적 Network Test |

관리 접속을 Session Manager로 통일하면 `app-sg`의 SSH Rule을 제거할 수 있습니다.

## 7. EC2 배치와 관리 경로

---

| EC2 | Subnet | Public IPv4 | Security Group |
| --- | --- | --- | --- |
| Bastion 또는 Web | `public-subnet-a` | 활성화 | `bastion-sg` 또는 `web-sg` |
| Application A | `private-subnet-a` | 비활성화 | `app-sg` |
| Application B | `private-subnet-b` | 비활성화 | `app-sg` |

Private EC2에는 `AmazonSSMManagedInstanceCore` 권한을 가진 Instance Profile을 연결하고 SSM Agent와 Outbound 경로를 준비하면 Session Manager로 접속할 수 있습니다.

```bash
# [LOCAL]
aws ssm start-session \
  --target <PRIVATE_INSTANCE_ID>
```

SSH 경로를 학습해야 한다면 Private Key를 Bastion에 복사하지 않고 Local Client에서 ProxyJump를 사용합니다.

```bash
# [LOCAL]
ssh -i <PRIVATE_KEY_PATH> \
  -J ubuntu@<BASTION_PUBLIC_IP> \
  ubuntu@<PRIVATE_INSTANCE_IP>
```

Amazon DocumentDB 같은 VPC 내부 Managed Database는 Public Internet에 직접 노출하지 않습니다. 관리 접속이 필요하면 Session Manager Port Forwarding 또는 제한된 관리 Host를 사용하고 Database Security Group에는 Application이나 관리 Host의 Security Group만 허용합니다.

## 8. Network 검증

---

### 8.1 Public EC2 접속

Public EC2의 Public IPv4와 Security Group을 확인한 뒤 제한된 관리 주소에서 접속합니다.

```bash
# [LOCAL]
ssh -i <PRIVATE_KEY_PATH> \
  ubuntu@<PUBLIC_INSTANCE_IP>
```

### 8.2 VPC 내부 경로

`[PUBLIC-EC2]`에서 Private EC2의 Private IP로 ICMP 또는 Application Port를 확인합니다. ICMP는 양쪽 Security Group에서 필요한 경우에만 허용합니다.

```bash
# [PUBLIC-EC2]
ping -c 3 <PRIVATE_INSTANCE_IP>

nc -vz <PRIVATE_INSTANCE_IP> <APP_PORT>
```

### 8.3 Private Subnet Outbound

`[PRIVATE-EC2]`에서 HTTPS 요청으로 NAT Gateway 경로를 확인합니다.

```bash
# [PRIVATE-EC2]
curl --fail --silent --show-error \
  https://checkip.amazonaws.com
```

출력 주소가 같은 Zone의 NAT Gateway Elastic IP인지 확인합니다. 실패하면 다음 순서로 점검합니다.

1. Private Subnet의 Default Route가 같은 Zone의 NAT Gateway를 가리키는지 확인합니다.

2. NAT Gateway가 Public Subnet에 있고 `Available` 상태인지 확인합니다.

3. Public Subnet Route Table에 Internet Gateway Default Route가 있는지 확인합니다.

4. EC2 Security Group의 Outbound와 두 Subnet의 Network ACL을 확인합니다.

5. VPC Flow Logs로 Reject Traffic을 확인합니다.

## 9. Resource 정리

---

NAT Gateway와 Public IPv4에는 비용이 발생할 수 있습니다. 실습 후 다음 순서로 제거합니다.

1. EC2 Instance를 종료합니다.

2. NAT Gateway 두 개를 삭제하고 해제 가능한 Elastic IP를 Release합니다.

3. Custom Route Table과 Security Group을 삭제합니다.

4. Internet Gateway를 VPC에서 Detach한 뒤 삭제합니다.

5. Subnet과 VPC를 삭제합니다.

> **최종 정리**
> - Public Subnet은 Internet Gateway Route로 결정되고 Public IPv4가 있어야 IPv4 Internet 통신이 가능합니다.
>
> - NAT Gateway는 Public Subnet에 만들고 Private Subnet의 Default Route Target으로 사용합니다.
>
> - 운영 환경에서는 Zone별 NAT Gateway를 배치해 장애 범위와 Cross-AZ Traffic을 줄입니다.
>
> - Private EC2 관리에는 Session Manager를 우선하고 SSH가 필요하면 Source 주소와 경로를 제한합니다.
>
> - Route는 경로를 결정하고 Security Group과 Network ACL은 허용 범위를 결정합니다.

## 참고 자료

---

- [Private Subnet과 NAT Gateway VPC 예제](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-example-private-subnets-nat.html)

- [NAT Gateway 문제 해결](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html)

- [VPC Route Table 구성](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)

- [Session Manager 시작](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html)
