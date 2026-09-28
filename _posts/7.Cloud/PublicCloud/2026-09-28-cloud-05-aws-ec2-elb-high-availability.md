---
title: Amazon EC2와 ELB 기반 고가용성 구성
description: EC2 Instance 생성과 SSH 접속, Security Group, Elastic IP, ALB와 Auto Scaling Group을 연결해 여러 Availability Zone에 분산된 고가용성 구조를 구성합니다
date: 2026-09-28
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> Amazon EC2는 OS, Compute, Network와 Storage를 직접 구성할 수 있는 Virtual Server Service입니다. 이 글에서는 EC2 Instance를 생성해 Apache Web Server를 배포하고, Security Group과 Network ACL의 차이를 확인합니다. 이어서 ALB, Target Group과 Auto Scaling Group을 연결해 여러 Availability Zone에 Workload를 분산하고 장애에 대응하는 흐름을 정리합니다.

## 1. Amazon EC2의 구조

---

Amazon EC2(Elastic Compute Cloud)는 Virtual Server인 Instance를 제공하는 Service입니다. Instance를 생성할 때 OS Image, CPU·Memory 조합, Network, 인증 수단, Storage와 Firewall을 함께 선택합니다.

### 1.1 주요 구성 요소

| 구성 요소 | 역할 |
| --- | --- |
| Instance | AWS에서 실행되는 Virtual Server입니다. |
| AMI | OS와 Software 구성을 포함하는 Instance 생성 Template입니다. |
| Instance Type | vCPU, Memory, Network와 가속기의 조합입니다. |
| Key Pair | Linux Instance의 SSH 공개키 인증에 사용합니다. AWS에는 공개키가 저장되고 사용자가 Private Key를 보관합니다. |
| EBS Volume | Root File System과 Data Disk에 사용하는 Block Storage입니다. |
| Security Group | ENI에 적용되는 Stateful Virtual Firewall입니다. |
| Elastic IP | Resource에 연결할 수 있는 고정 Public IPv4 Address입니다. |

```text
AMI + Instance Type + VPC/Subnet
              +
Key Pair + Security Group + EBS
              ↓
         EC2 Instance
```

### 1.2 EC2가 적합한 경우

- OS와 Middleware를 직접 제어해야 하는 경우입니다.

- 기존 Application을 Virtual Server 형태로 이전하는 경우입니다.

- 특수한 CPU, Memory, GPU 또는 Local Storage 구성이 필요한 경우입니다.

- AMI와 Auto Scaling을 이용해 동일한 Server를 반복 생성해야 하는 경우입니다.

운영 작업을 최소화하려면 ECS, EKS, Lambda나 Managed Database처럼 더 높은 수준의 Service가 적합할 수 있습니다. 변화가 거의 없는 단일 Server라도 EC2를 사용할 수 있지만 Backup, Patch, Monitoring과 장애 복구를 사용자가 책임져야 합니다.

## 2. EC2 비용을 구성하는 항목

---

EC2 비용은 Instance 실행 비용 하나로 끝나지 않습니다.

| 항목 | 과금 기준과 주의 사항 |
| --- | --- |
| Instance | Instance Type, 구매 방식, OS와 실행 시간에 따라 계산됩니다. Stop 상태에서는 일반적인 Compute 요금이 중지되지만 일부 연결 Resource 비용은 계속 발생합니다. |
| EBS | 실제 File 사용량이 아니라 Provisioning한 Volume 용량과 유형, IOPS·Throughput 등에 따라 비용이 발생합니다. Instance를 Stop해도 과금됩니다. |
| Snapshot과 AMI | Snapshot에 저장된 Data와 AMI가 참조하는 Snapshot에 비용이 발생할 수 있습니다. |
| Data Transfer | Internet Outbound, AZ 간 전송과 Service 경로에 따라 비용이 달라집니다. Inbound가 항상 유료인 것은 아니지만 경로별 가격을 확인해야 합니다. |
| Public IPv4 | 일반 Public IPv4와 Elastic IP 모두 사용 중이거나 유휴 상태인지와 관계없이 시간당 비용이 발생합니다. |
| Load Balancer | Load Balancer 실행 시간과 처리 용량 단위에 따라 비용이 발생합니다. |

2024년 2월 1일부터 AWS가 제공하는 Public IPv4 Address는 연결 여부와 관계없이 과금됩니다. `Elastic IP 한 개는 무료`라는 이전 기준을 적용하면 예상하지 못한 비용이 생길 수 있습니다.

## 3. EC2 Instance 생성

---

### 3.1 생성 전에 결정할 값

| 항목 | 예제 | 확인할 내용 |
| --- | --- | --- |
| Region | `ap-northeast-2` | 비용, 지연 시간, Service 지원 범위 |
| AMI | Ubuntu Server LTS | Architecture와 지원 기간 |
| Instance Type | Free Tier 또는 실습에 적합한 Type | vCPU, Memory, Architecture와 가격 |
| VPC·Subnet | 실습용 Public Subnet | Internet Gateway와 Route Table |
| Public IP | 실습에 필요한 경우만 활성화 | IPv4 비용과 노출 범위 |
| Key Pair | ED25519 또는 RSA | Private Key 보관 위치와 Region |
| Security Group | SSH와 HTTP 최소 허용 | Source CIDR과 Port |
| EBS | Root Volume | 용량, 유형, 암호화와 종료 시 삭제 여부 |

Instance Family는 세대에 따라 계속 추가됩니다. `M2`, `CR1`처럼 오래된 Type을 기준으로 선택하지 않고 Console의 Instance Type Selector와 현재 EC2 문서를 확인합니다.

| Family 예 | 목적 |
| --- | --- |
| `t`, `m` | Burstable 또는 범용 Workload |
| `c` | Compute 중심 Workload |
| `r`, `x` | Memory 중심 Workload |
| `g`, `p` | GPU·Accelerated Computing |
| `i`, `d` | Storage 중심 Workload |

### 3.2 AMI 선택

AMI(Amazon Machine Image)는 Instance를 시작하는 Template입니다. 다음 출처를 구분해 선택합니다.

- **Quick Start**: AWS가 제공하는 공식 Image입니다.

- **Marketplace**: Vendor가 제공하는 유료 또는 무료 Image입니다. Software 사용료가 Instance 비용에 추가될 수 있습니다.

- **Community AMI**: 제3자가 공개한 Image입니다. Publisher, Owner ID, Update 일자와 보안 상태를 검증해야 합니다.

- **My AMI**: 자신의 Account에서 생성했거나 공유받은 Image입니다.

CPU Architecture가 `x86_64`인지 `arm64`인지 확인하고 선택한 Instance Type이 같은 Architecture를 지원하는지 확인합니다.

### 3.3 Key Pair 생성

Key Pair를 생성하면 AWS는 Public Key를 Instance에 배치하고 Private Key File을 한 번만 Download할 수 있게 제공합니다. Download한 Private Key는 다시 받을 수 없으므로 안전하게 보관합니다.

macOS와 Linux에서는 Permission을 제한합니다.

```bash
chmod 400 <KEY_FILE>.pem
```

Key Pair는 Region Resource입니다. 서울 Region에서 생성한 Key Pair가 다른 Region에 자동으로 복제되지는 않습니다.

### 3.4 Security Group 생성

처음에는 다음과 같이 최소한으로 허용합니다.

| 방향 | Protocol | Port | Source·Destination | 목적 |
| --- | --- | --- | --- | --- |
| Inbound | TCP | 22 | 관리자 Public IP `/32` | SSH |
| Inbound | TCP | 80 | 실습에서만 `0.0.0.0/0`, 운영에서는 ALB Security Group | HTTP |
| Outbound | 필요한 Protocol | 필요한 Port | 업무에 필요한 대상 | Package Download와 외부 연결 |

SSH Port를 `0.0.0.0/0`에 공개하지 않습니다. 고정 Source IP를 사용할 수 없다면 AWS Systems Manager Session Manager 또는 VPN·Bastion 구성을 검토합니다.

### 3.5 EBS 설정

EBS는 EC2 Instance에 연결하는 Block Storage입니다. Root Volume 설정에서 다음 항목을 확인합니다.

- Volume Type과 용량이 Workload에 적합한지 확인합니다.

- 암호화를 활성화하고 사용할 KMS Key를 확인합니다.

- `Delete on termination`이 활성화되면 Instance 종료 시 Root Volume도 삭제됩니다.

- 보존해야 하는 Data는 별도 Volume, Snapshot 또는 외부 Storage에 Backup합니다.

Instance를 Stop해도 EBS Volume은 남아 있으며 비용이 계속 발생합니다.

## 4. EC2에 연결하기

---

### 4.1 기본 계정 확인

SSH 계정은 선택한 AMI에 따라 다릅니다.

| AMI | 일반적인 기본 계정 |
| --- | --- |
| Ubuntu | `ubuntu` |
| Amazon Linux | `ec2-user` |

AMI 공급자가 다른 경우 해당 Image 문서에서 계정을 확인합니다.

### 4.2 Local Terminal에서 SSH 연결

Management Console의 Instance 상세 화면에서 `연결`을 선택하면 EC2 Instance Connect 등 해당 Instance에 사용할 수 있는 연결 방식을 확인할 수 있습니다. Browser 연결의 지원 여부는 AMI, Network 경로, IAM 권한과 Instance Connect 구성에 따라 달라집니다.

다음 명령은 Private Key와 Public IPv4 또는 Public DNS를 사용해 접속합니다.

```bash
ssh -i <KEY_FILE>.pem <USER>@<PUBLIC_IP_OR_DNS>
```

연결되지 않으면 다음 순서로 확인합니다.

1. Instance가 `running`이고 Status Check를 통과했는지 확인합니다.

2. Public Subnet의 Route Table이 Internet Gateway로 연결되는지 확인합니다.

3. Instance에 Public IPv4 또는 Elastic IP가 있는지 확인합니다.

4. Security Group의 TCP 22 Source가 현재 Client IP를 허용하는지 확인합니다.

5. Network ACL이 SSH 응답에 필요한 Ephemeral Port를 차단하지 않는지 확인합니다.

6. AMI의 기본 계정과 Key Pair가 올바른지 확인합니다.

### 4.3 Password SSH를 사용하지 않는 이유

Private Key File을 매번 지정하기 번거롭다는 이유로 `PasswordAuthentication yes`를 활성화하면 Internet에 노출된 SSH Server가 Password 추측 공격을 받기 쉬워집니다. `/etc/sudoers`에 쓰기 권한을 추가하는 방식도 파일 훼손과 권한 상승 위험이 있습니다.

다음 방법을 우선 사용합니다.

- `~/.ssh/config`에 Host, User와 `IdentityFile`을 등록합니다.

- SSH Agent에 Private Key를 등록합니다.

- 팀 단위 접근은 EC2 Instance Connect 또는 Systems Manager Session Manager를 검토합니다.

- 별도 사용자의 `sudo` 권한이 필요하면 `/etc/sudoers.d/<USER>`를 `visudo -f`로 검증합니다.

```sshconfig
Host aws-lab
  HostName <PUBLIC_IP_OR_DNS>
  User ubuntu
  IdentityFile ~/.ssh/<KEY_FILE>.pem
  IdentitiesOnly yes
```

등록 후에는 다음과 같이 접속합니다.

```bash
ssh aws-lab
```

## 5. Apache Web Server 배포

---

다음 명령은 Ubuntu EC2 Instance에서 실행합니다.

```bash
sudo apt update
sudo apt install -y apache2
sudo systemctl enable --now apache2
```

`enable --now`는 Boot 시 자동 시작을 등록하고 현재 Service도 즉시 시작합니다.

Service와 Local HTTP 응답을 확인합니다.

```bash
systemctl is-active apache2
curl -I http://127.0.0.1
```

기본 Document Root는 `/var/www/html`입니다. 다음 내용을 `/var/www/html/index.html`에 작성합니다.

```html
<!doctype html>
<html lang="ko">
  <head>
    <meta charset="utf-8">
    <title>AWS EC2</title>
  </head>
  <body>
    <h1>AWS EC2</h1>
  </body>
</html>
```

외부 Client에서 접속하려면 Instance Security Group에 TCP 80 Inbound Rule이 필요합니다.

```text
http://<PUBLIC_IP>
```

외부 접속이 실패하면 Apache가 실행 중인지, TCP 80 Rule과 Source CIDR이 올바른지, Public Subnet Route와 Public IP가 있는지 순서대로 확인합니다.

## 6. Security Group과 Network ACL

---

Security Group과 Network ACL은 모두 Traffic을 제어하지만 적용 위치와 상태 처리 방식이 다릅니다.

| 구분 | Security Group | Network ACL |
| --- | --- | --- |
| 적용 범위 | Elastic Network Interface | Subnet |
| 상태 처리 | Stateful | Stateless |
| Rule | Allow Rule만 사용 | Allow와 Deny Rule 사용 |
| 응답 Traffic | 허용된 요청의 응답을 자동 허용 | Inbound와 Outbound를 각각 허용해야 함 |

Security Group 개수와 Rule 수는 Service Quota의 영향을 받습니다. 고정된 숫자를 외우기보다 현재 Account의 Service Quotas에서 확인합니다.

운영 환경에서 ALB와 EC2를 함께 사용할 때는 Security Group을 다음과 같이 연결합니다.

```text
Internet
  ↓ TCP 80 또는 443
ALB Security Group
  ↓ Application Port
EC2 Security Group
  ↓
EC2 Instance
```

EC2 Security Group의 Source를 `0.0.0.0/0`으로 열지 않고 ALB Security Group ID로 제한하면 Client가 Load Balancer를 우회해 Instance에 직접 접근하는 것을 막을 수 있습니다.

## 7. Elastic IP와 Public IPv4

---

자동 할당된 EC2 Public IPv4는 Instance를 Stop한 뒤 Start하면 바뀔 수 있습니다. Elastic IP는 Account에 할당되어 Resource에 다시 연결할 수 있는 고정 Public IPv4입니다.

한 Elastic IP를 여러 Instance에 동시에 연결하는 것은 아닙니다. 장애나 교체 상황에서 연결 대상을 다른 Network Interface 또는 Instance로 재연결할 수 있습니다.

다음 사항을 확인합니다.

- Elastic IP는 Region Resource입니다.

- Instance를 삭제해도 Elastic IP를 Release하지 않으면 Account에 남습니다.

- 연결 중인 Public IPv4와 사용하지 않는 Elastic IP 모두 비용이 발생합니다.

- 일반적인 Web Service는 고정 IP가 꼭 필요하지 않다면 ALB DNS, Route 53과 IPv6 사용을 검토합니다.

## 8. Elastic Load Balancing

---

Elastic Load Balancing(ELB)은 여러 Target의 상태를 확인하고 정상 Target으로 Traffic을 분산하는 Managed Service입니다.

Load Balancer의 기반 장비를 사용자가 직접 증설하지는 않습니다. AWS가 Service 용량을 관리하지만, Load Balancer 뒤의 Target Capacity와 Service Quota, 갑작스러운 Traffic 증가에 대한 Application 성능은 별도로 설계하고 검증해야 합니다.

### 8.1 Load Balancer 종류

| 종류 | 주요 계층·Protocol | 사용 목적 |
| --- | --- | --- |
| ALB | Layer 7, HTTP·HTTPS | Host, Path, Header 등의 Application Routing |
| NLB | Layer 4, TCP·UDP·TLS | 낮은 지연 시간, 고정 IP 요구와 대규모 Connection |
| GWLB | Layer 3 Gateway와 GENEVE | Firewall, IDS·IPS와 같은 Virtual Network Appliance 연결 |
| CLB | Legacy, TCP·SSL/TLS·HTTP·HTTPS | 기존 System 호환을 위한 이전 세대 Load Balancer |

새로운 HTTP·HTTPS Application은 일반적으로 ALB를 사용합니다. CLB는 기존 Workload를 위한 이전 세대 Service이므로 신규 구성에서는 ALB 또는 NLB를 우선 검토합니다.

### 8.2 Target Group

Target Group은 Load Balancer가 요청을 전달할 대상과 Health Check 설정을 관리합니다.

```text
Listener
  ↓ Rule 평가
Target Group
  ├── Target A: healthy
  ├── Target B: healthy
  └── Target C: unhealthy
```

Target Type은 Load Balancer 종류에 따라 지원 범위가 다릅니다.

| Target Type | 예시 |
| --- | --- |
| Instance | EC2 Instance ID를 등록합니다. |
| IP | VPC 또는 연결된 On-premises의 Private IP를 등록합니다. |
| Lambda | ALB가 Lambda Function을 호출합니다. |
| ALB | NLB의 Target으로 ALB를 연결하는 구성에 사용합니다. |

Health Check가 실패한 Target에는 정상 상태로 돌아올 때까지 새 요청을 전달하지 않습니다.

### 8.3 ALB 구성 순서

1. 서로 다른 AZ에 Public Subnet을 준비합니다.

2. Internet에서 TCP 80 또는 443을 허용하는 ALB Security Group을 생성합니다.

3. Application Port와 Health Check Path를 지정한 Target Group을 생성합니다.

4. EC2 Instance를 Target Group에 등록하고 `healthy` 상태인지 확인합니다.

5. Internet-facing ALB를 생성하고 최소 두 AZ의 Subnet을 선택합니다.

6. Listener Rule에서 요청을 Target Group으로 전달합니다.

7. ALB DNS Name으로 접속해 응답을 확인합니다.

EC2 Security Group은 Application Port의 Source로 ALB Security Group만 허용합니다.

### 8.4 503과 Timeout 진단

ALB에서 HTTP 503이 발생했다고 Target Group을 삭제하고 다시 만드는 것은 원인 해결 절차가 아닙니다. 503은 Target Group에 등록된 Target이 없거나 요청을 받을 준비가 된 Target이 부족할 때 발생할 수 있습니다.

다음 항목을 확인합니다.

```text
Target 등록 여부
  ↓
Target Health 상태와 실패 사유
  ↓
Health Check Path·Port·Success Code
  ↓
ALB SG → EC2 SG 통신 허용
  ↓
Application Process와 Listening Port
```

ALB의 기본 Idle Timeout은 60초입니다. Target 연결 실패, Target 응답 지연과 Idle Timeout 초과는 502 또는 504처럼 다른 상태 코드로 나타날 수 있으므로 Access Log, CloudWatch Metric과 Target Health 사유를 함께 확인합니다.

### 8.5 ELB 비용

ALB 비용은 Load Balancer 사용 시간과 LCU(Load Balancer Capacity Unit)를 기준으로 계산됩니다. LCU는 다음 차원 중 해당 시간에 가장 많이 사용한 항목을 기준으로 산정되므로 Connection 수만 확인해서는 비용을 예측하기 어렵습니다.

- 초당 새 Connection 수입니다.

- 분당 활성 Connection 수입니다.

- 시간당 처리한 Byte입니다.

- 초당 Rule 평가 수입니다.

Protocol, Certificate Key 크기, Lambda Target 사용 여부 등에 따라 LCU 기준이 달라질 수 있으므로 실제 비용은 [Elastic Load Balancing 요금](https://aws.amazon.com/elasticloadbalancing/pricing/)에서 확인합니다.

## 9. 고가용성과 내결함성

---

### 9.1 고가용성

고가용성(High Availability)은 장애가 발생해도 Service를 사용할 수 있는 시간을 목표 수준 이상으로 유지하는 설계입니다. 장애 감지와 복구 과정에서 짧은 중단이나 성능 저하가 발생할 수 있습니다.

가용성 목표에 따른 연간 허용 중단 시간의 근삿값은 다음과 같습니다.

| 가용성 | 연간 허용 중단 시간 |
| --- | --- |
| 99% | 약 3일 15시간 |
| 99.9% | 약 8시간 46분 |
| 99.99% | 약 52분 34초 |
| 99.999% | 약 5분 15초 |

실제 SLO는 측정 구간, 계획된 점검 포함 여부와 Error Budget 기준을 함께 정의해야 합니다.

### 9.2 내결함성

내결함성(Fault Tolerance)은 일부 구성 요소가 고장 나더라도 Service가 중단되지 않고 필요한 기능과 용량을 유지하도록 설계하는 성질입니다. 고가용성보다 더 많은 중복 Resource와 비용이 필요할 수 있습니다.

| 구분 | 고가용성 | 내결함성 |
| --- | --- | --- |
| 목표 | 빠른 복구를 통해 가용 시간 유지 | 구성 요소 장애 중에도 Service 지속 |
| 장애 영향 | 짧은 중단이나 성능 저하를 허용할 수 있음 | 필요한 기능과 용량을 계속 유지하도록 설계 |
| 주요 수단 | Health Check, Failover, Auto Recovery, Multi-AZ | Active-Active, N+1·N+M 중복, Data Replication |

내결함성은 계층마다 다른 방식으로 구현합니다.

- Hardware 계층에서는 RAID와 이중 Power Supply로 Disk 또는 전원 장치 장애를 견딥니다.

- Database 계층에서는 Replication으로 Data 복제본을 유지하고 장애 시 대체 Node로 전환합니다.

- Application 계층에서는 Error Handling과 Circuit Breaker로 반복해서 실패하는 의존성을 격리하고 장애 전파를 제한합니다.

이 기능들이 존재한다고 System 전체가 자동으로 내결함성을 갖는 것은 아닙니다. 장애 범위, 복제 지연, 전환 조건과 남은 Capacity를 함께 검증해야 합니다.

### 9.3 핵심 설계 원칙

- **Single Point of Failure 제거**: 한 구성 요소의 장애가 전체 Service 중단으로 이어지지 않도록 합니다.

- **중복과 분산**: 여러 Instance, AZ와 Data 복제본을 사용합니다.

- **Health Check와 자동 복구**: 장애를 감지하고 Traffic을 정상 Target으로 전환하거나 Instance를 교체합니다.

- **Capacity 여유**: 하나의 AZ가 사라져도 남은 AZ에서 필요한 Traffic을 처리할 수 있게 설계합니다.

### 9.4 Instance 6대가 필요한 System

정상 운영에 Instance 6대의 처리 용량이 반드시 필요하고 AZ 하나의 전체 장애를 가정합니다.

| 배치 | AZ 하나 장애 후 남는 Instance | 6대 용량 유지 |
| --- | ---: | --- |
| 2개 AZ에 3대씩 | 3대 | 불가능 |
| 3개 AZ에 2대씩 | 4대 | 불가능 |
| 2개 AZ에 6대씩 | 6대 | 가능 |
| 3개 AZ에 3대씩 | 6대 | 가능 |
| 1개 AZ에 6대 | 0대 | 불가능 |

앞의 두 분산 방식도 한 AZ 장애 후 일부 Service를 계속할 수 있으므로 가용성 향상에는 도움이 됩니다. 그러나 `항상 6대가 필요하다`는 조건에서는 필요한 Capacity를 유지하지 못합니다. 이 조건에서 AZ 하나의 장애를 견디는 배치는 `2개 AZ에 6대씩` 또는 `3개 AZ에 3대씩`입니다.

## 10. Auto Scaling Group

---

Auto Scaling Group(ASG)은 Launch Template을 이용해 EC2 Instance 수와 배치를 관리합니다. Load Balancer와 연결하면 새 Instance를 Target Group에 등록하고 비정상 Instance를 교체할 수 있습니다.

```text
CloudWatch Metric 또는 일정
            ↓
      Scaling Policy
            ↓
   Auto Scaling Group
      ├── AZ A Instance
      ├── AZ B Instance
      └── AZ C Instance
            ↓
        Target Group
            ↓
            ALB
```

### 10.1 Capacity 설정

| 값 | 역할 |
| --- | --- |
| Minimum Capacity | Group이 유지할 최소 Instance 수입니다. |
| Desired Capacity | 현재 Group이 유지하려는 Instance 수입니다. |
| Maximum Capacity | Scale-out할 수 있는 최대 Instance 수입니다. |

Desired Capacity는 Minimum과 Maximum 사이에 있어야 합니다. 여러 AZ의 Subnet을 선택하면 ASG가 Instance를 분산하도록 구성할 수 있습니다.

### 10.2 Scaling Policy

| 정책 | 동작 |
| --- | --- |
| Target Tracking | 평균 CPU 사용률 같은 Metric을 목표값에 맞추도록 Capacity를 조절합니다. |
| Step Scaling | Alarm 크기에 따라 정해진 단계만큼 Capacity를 변경합니다. |
| Scheduled Scaling | 예상 가능한 행사나 업무 시간에 맞춰 Capacity를 변경합니다. |

Auto Scaling은 Instance 생성과 Application 준비에 시간이 필요합니다. 갑작스러운 Traffic 증가가 시작 속도보다 빠르면 장애가 발생할 수 있으므로 다음 항목을 함께 고려합니다.

- 충분한 Minimum·Desired Capacity를 유지합니다.

- Warm Pool 또는 빠른 Bootstrap을 검토합니다.

- Metric과 Cooldown·Instance Warmup을 Workload에 맞게 설정합니다.

- Load Test로 Scale-out 속도와 최대 Capacity를 검증합니다.

- CloudFront, Cache와 Queue로 급격한 부하를 흡수합니다.

## 11. Snapshot과 AMI

---

EBS Snapshot은 특정 시점의 EBS Volume Data를 보존합니다. 첫 Snapshot 이후에는 변경된 Block을 기준으로 증분 저장하지만, 사용자는 각 Snapshot을 독립적인 복구 지점처럼 사용할 수 있습니다.

AMI는 EC2 Instance를 시작하기 위한 Template입니다. AMI에는 다음 정보가 포함됩니다.

- Root Volume과 추가 Volume을 구성하는 Block Device Mapping입니다.

- AMI가 참조하는 하나 이상의 EBS Snapshot입니다.

- Architecture, Root Device와 Virtualization 관련 Metadata입니다.

- Launch Permission입니다.

| 구분 | EBS Snapshot | AMI |
| --- | --- | --- |
| 목적 | Volume Data Backup과 복원 | 동일한 구성의 Instance 생성 |
| 생성 기준 | EBS Volume | EC2 Instance 또는 Snapshot 구성 |
| 생성 결과 | 새 EBS Volume의 원본 | 새 EC2 Instance의 Template |
| 주요 사용 | Data 복구, Volume 복제 | Auto Scaling, 표준 Server Image, 반복 배포 |

AMI를 생성할 때 File System과 Application Data의 일관성이 필요하면 Application을 정지하거나 Flush한 뒤 Image를 생성합니다. `No reboot`를 선택하면 중단은 줄일 수 있지만 File System과 Application 상태의 일관성을 별도로 보장해야 합니다.

## 12. 전체 동작 흐름

---

```text
Client
  ↓ DNS
Route 53
  ↓
Application Load Balancer
  ↓ Listener Rule
Target Group
  ├── AZ A의 EC2 Instance
  ├── AZ B의 EC2 Instance
  └── AZ C의 EC2 Instance
          ↑
   Auto Scaling Group
          ↑
Launch Template + AMI
```

1. Route 53이 Application Domain을 ALB로 연결합니다.

2. ALB Listener가 요청을 받고 Rule에 따라 Target Group을 선택합니다.

3. Target Group은 Health Check를 통과한 EC2 Instance로 요청을 전달합니다.

4. ASG는 여러 AZ에 Desired Capacity만큼 Instance를 유지합니다.

5. Instance 장애나 Scaling Policy가 감지되면 ASG가 AMI와 Launch Template으로 Instance를 생성하거나 제거합니다.

6. EBS Snapshot과 AMI는 Data 복구와 반복 가능한 Instance 생성에 사용합니다.

## 13. 실습 후 비용 정리

---

실습을 마친 뒤 Instance만 종료하면 관련 비용이 모두 사라지는 것은 아닙니다. 다음 Resource를 확인합니다.

- Auto Scaling Group의 Minimum과 Desired Capacity를 0으로 조정하거나 실습 Group을 삭제합니다.

- Load Balancer를 삭제합니다.

- 사용하지 않는 Target Group과 Launch Template을 정리합니다.

- EC2 Instance를 종료합니다.

- 남겨진 EBS Volume과 Snapshot, AMI가 참조하는 Snapshot을 확인합니다.

- Elastic IP를 연결 해제한 뒤 Release합니다.

- CloudWatch Log Group과 Alarm 보존 여부를 확인합니다.

- Cost Explorer와 Billing Dashboard에서 Public IPv4, EC2, EBS와 ELB 비용을 확인합니다.

## 참고 자료

---

- [Amazon EC2 User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)

- [Amazon VPC Public IPv4 요금](https://aws.amazon.com/vpc/pricing/)

- [Application Load Balancer 문제 해결](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-troubleshooting.html)

- [Elastic Load Balancing 요금](https://aws.amazon.com/elasticloadbalancing/pricing/)
