---
title: AWS Site-to-Site VPN Architecture와 EC2 Router 실습
description: Customer Gateway, Virtual Private Gateway와 두 IPsec Tunnel의 역할을 이해하고 서로 다른 CIDR의 두 VPC로 On-Premise 연결을 모의 구성합니다
date: 2026-10-08
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> AWS Site-to-Site VPN은 On-Premise Customer Gateway와 AWS 측 Virtual Private Gateway 또는 Transit Gateway 사이에 두 IPsec Tunnel을 구성합니다. Static Route와 BGP Dynamic Route의 차이를 구분하고, 실제 Router 대신 Public EC2에 StrongSwan을 설치해 On-Premise Network를 모의합니다. 두 VPC는 겹치지 않는 CIDR를 사용하고 Tunnel Secret은 문서나 Repository에 저장하지 않습니다.

## 1. Site-to-Site VPN 구성 요소

---

기본 연결 구조는 다음과 같습니다.

```text
On-Premise Network
  ↓
Customer Gateway Device
  ⇅ IPsec Tunnel 1·2
Virtual Private Gateway 또는 Transit Gateway
  ↓
AWS VPC Private Network
```

| 구성 요소 | 역할 |
| --- | --- |
| Customer Gateway Device | On-Premise Router 또는 Firewall에서 VPN Tunnel 종료 |
| Customer Gateway Resource | Device의 Public IP와 BGP ASN 등을 AWS에 표현 |
| Site-to-Site VPN Connection | AWS와 Customer Gateway 사이의 두 Tunnel |
| Virtual Private Gateway | 하나의 VPC에 Attach하는 AWS 측 VPN 종단점 |
| Transit Gateway | 여러 VPC와 On-Premise Network를 Hub 형태로 연결 |

Customer Gateway Device에는 Internet에서 도달 가능한 고정 Public IP가 필요합니다. Device가 NAT 뒤에 있고 NAT Traversal을 지원하면 AWS Customer Gateway Resource에는 NAT Device의 Public IP를 사용합니다.

## 2. 두 Tunnel과 보안 Protocol

---

AWS Site-to-Site VPN Connection은 고가용성을 위해 두 Tunnel을 제공합니다. Customer Gateway Device에 두 Tunnel을 모두 구성해야 AWS 측 Tunnel Endpoint 유지보수나 장애 시 보조 경로를 사용할 수 있습니다.

두 Tunnel을 단순히 Primary와 Secondary로 고정하지 않습니다. Route 방식과 Target Gateway에 따라 Traffic 경로가 달라질 수 있으므로 AWS가 제공한 두 Tunnel 구성과 Route 우선순위를 함께 적용합니다.

| Protocol | 역할 |
| --- | --- |
| IKE | 양 끝의 Security Association 협상과 Key 교환 |
| IPsec ESP | IP Packet Payload의 기밀성과 무결성 보호 |
| NAT-T | NAT 환경에서 IPsec Traffic을 UDP 4500으로 전달 |

방화벽에서는 AWS가 생성한 Tunnel Outside IP를 Source 또는 Destination으로 제한하고 필요한 IKE, NAT-T와 ESP Traffic만 허용합니다.

## 3. Static Route와 BGP Dynamic Route

---

| 구분 | Static Route | BGP Dynamic Route |
| --- | --- | --- |
| 경로 등록 | On-Premise CIDR를 수동 등록 | BGP Peer가 경로 광고 |
| Network 변경 | Route를 수동 갱신 | 광고한 경로를 동적으로 반영 |
| 장애 감지 | Device의 별도 Health Check 필요 | BGP Liveness가 Failover 판단 지원 |
| 적용 | 소규모 Lab, BGP 미지원 Device | 변경이 잦거나 복수 경로를 가진 환경 |

BGP를 지원하는 Device에는 Dynamic Routing을 우선 검토합니다. Static Routing을 선택하면 VPN Connection 생성 시 AWS에 광고할 On-Premise Prefix를 직접 입력합니다.

Customer Gateway ASN은 지원되는 Private ASN 범위에서 선택하고 AWS 측 Gateway ASN과 다르게 설정합니다. 단순히 특정 기본 번호를 유지하는 것이 Static Routing의 조건은 아닙니다.

## 4. Packet 왕복 흐름

---

On-Premise Client가 VPC EC2에 접근하는 흐름은 다음과 같습니다.

```text
On-Premise Client
  ↓ On-Premise Route Table
Customer Gateway에서 IPsec 암호화
  ↓ Internet
AWS VPN Tunnel Endpoint에서 복호화
  ↓ Virtual Private Gateway
VPC Route Table
  ↓ Security Group·Network ACL
Target EC2
```

응답은 반대 경로로 돌아가야 합니다. VPC에 On-Premise CIDR Route가 있어도 On-Premise Subnet이 VPC CIDR를 Customer Gateway로 보내지 않으면 통신할 수 없습니다.

## 5. Lab 구성과 주의 사항

---

실제 On-Premise Router가 없는 환경에서는 두 VPC 중 하나를 On-Premise Network처럼 사용합니다. 이 구성은 VPN 학습용이며 실제 Data Center의 Firewall, 회선과 장애 조건을 대체하지 않습니다.

```text
Simulated On-Premise VPC 10.10.0.0/16
├── Public Router Subnet 10.10.1.0/24
│   └── StrongSwan Router EC2 + Elastic IP
└── Private Client Subnet 10.10.11.0/24
    └── Client EC2

Cloud VPC 10.20.0.0/16
└── Private Workload Subnet 10.20.11.0/24
    └── Target EC2
```

두 VPC가 같은 CIDR를 사용하면 Route가 겹치므로 VPN 실습을 진행할 수 없습니다.

| 표기 | 실행 위치 | 작업 |
| --- | --- | --- |
| `[AWS-CONSOLE]` | AWS Management Console | VPC, VPN Resource, Route와 Security 구성 |
| `[ROUTER-EC2]` | On-Premise Simulation Router | StrongSwan, IP Forwarding과 Tunnel 구성 |
| `[ONPREM-CLIENT]` | Simulated On-Premise Private EC2 | Cloud VPC 연결 검증 |
| `[CLOUD-TARGET]` | Cloud VPC Private EC2 | Service와 반환 경로 검증 |

## 6. 두 VPC와 EC2 준비

---

### 6.1 Simulated On-Premise VPC

다음 Resource를 준비합니다.

| Resource | 값 |
| --- | --- |
| VPC | `onprem-vpc`, `10.10.0.0/16` |
| Router Subnet | `10.10.1.0/24`, Internet Gateway Route |
| Client Subnet | `10.10.11.0/24` |
| Router EC2 | Ubuntu, Elastic IP, Public Subnet |
| Client EC2 | Ubuntu, Public IPv4 없음 |

Client Subnet Route Table에는 Cloud VPC 경로를 Router EC2의 Network Interface로 추가합니다.

| Destination | Target |
| --- | --- |
| `10.20.0.0/16` | `<ROUTER_EC2_NETWORK_INTERFACE_ID>` |

Router EC2는 다른 Instance의 Packet을 전달해야 하므로 `Source/destination check`를 비활성화합니다.

### 6.2 Cloud VPC

다음 Resource를 준비합니다.

| Resource | 값 |
| --- | --- |
| VPC | `cloud-vpc`, `10.20.0.0/16` |
| Workload Subnet | `10.20.11.0/24` |
| Target EC2 | Ubuntu, Public IPv4 없음 |

Target EC2 Security Group은 Test Protocol을 `10.10.0.0/16`에서만 허용합니다. 관리 접속은 Session Manager를 사용합니다.

## 7. Router EC2 Network 설정

---

Router Security Group은 전체 Internet가 아니라 AWS가 할당한 두 VPN Tunnel Outside IP와 관리자 주소로 Source를 제한합니다.

| Protocol | Port 또는 Type | Source |
| --- | --- | --- |
| UDP | 500 | `<AWS_TUNNEL_1_OUTSIDE_IP>/32`, `<AWS_TUNNEL_2_OUTSIDE_IP>/32` |
| UDP | 4500 | `<AWS_TUNNEL_1_OUTSIDE_IP>/32`, `<AWS_TUNNEL_2_OUTSIDE_IP>/32` |
| ESP | Protocol 50 | Tunnel 구성에서 요구하는 경우 AWS Tunnel Endpoint |
| TCP | 22 | `<ADMIN_PUBLIC_IP>/32` |

`[ROUTER-EC2]`에서 StrongSwan을 설치하고 IP Forwarding을 영구 활성화합니다.

```bash
# [ROUTER-EC2]
sudo apt update
sudo apt install -y strongswan

sudo tee /etc/sysctl.d/99-vpn-router.conf >/dev/null <<'EOF'
net.ipv4.ip_forward=1
EOF

sudo sysctl --system
sysctl net.ipv4.ip_forward
```

`net.ipv4.ip_forward = 1`이 출력돼야 합니다.

## 8. AWS VPN Resource 구성

---

### 8.1 Virtual Private Gateway

`[AWS-CONSOLE]`에서 Virtual Private Gateway를 만들고 `cloud-vpc`에 Attach합니다.

### 8.2 Customer Gateway

Customer Gateway Resource에 다음 값을 설정합니다.

| 항목 | 값 |
| --- | --- |
| IP Address | Router EC2의 Elastic IP |
| Routing | 이 Lab에서는 Static |
| BGP ASN | Console이 요구하는 지원 범위의 예제 ASN |

Static Routing이어도 Customer Gateway Resource에는 ASN Field가 필요합니다. 이 값은 BGP Session을 구성하지 않으면 실제 경로 교환에 사용되지 않습니다.

### 8.3 Site-to-Site VPN Connection

Target Gateway로 Virtual Private Gateway를, Customer Gateway로 앞에서 만든 Resource를 선택합니다. Static IP Prefix에는 Simulated On-Premise VPC CIDR인 `10.10.0.0/16`을 입력합니다.

VPN이 생성되면 Router Software에 맞는 Sample Configuration을 Download합니다. File에는 Tunnel Outside IP와 Pre-Shared Key가 있으므로 Repository, Blog, Screenshot과 협업 Chat에 올리지 않습니다.

## 9. StrongSwan Tunnel 구성

---

AWS에서 Download한 Configuration의 IKE Version, Encryption, Integrity, Diffie-Hellman Group, Lifetime과 Tunnel별 Outside IP를 기준으로 StrongSwan을 설정합니다. 임의의 Algorithm으로 바꾸거나 두 Tunnel을 하나의 설정으로 합치지 않습니다.

`/etc/ipsec.secrets`에는 실제 값 대신 다음 구조가 들어갑니다.

```text
<CUSTOMER_GATEWAY_PUBLIC_IP> <AWS_TUNNEL_1_OUTSIDE_IP> : PSK "<VPN_TUNNEL_1_PRESHARED_KEY>"
<CUSTOMER_GATEWAY_PUBLIC_IP> <AWS_TUNNEL_2_OUTSIDE_IP> : PSK "<VPN_TUNNEL_2_PRESHARED_KEY>"
```

Secret File 권한을 제한합니다.

```bash
# [ROUTER-EC2]
sudo chown root:root /etc/ipsec.secrets
sudo chmod 600 /etc/ipsec.secrets
```

AWS Configuration File에 맞춰 `/etc/ipsec.conf`와 필요한 Tunnel별 설정을 작성한 뒤 StrongSwan을 다시 시작합니다.

```bash
# [ROUTER-EC2]
sudo systemctl restart strongswan-starter
sudo systemctl status strongswan-starter --no-pager
sudo ipsec statusall
```

두 Tunnel의 Security Association 상태를 각각 확인합니다. Download File의 실제 Secret과 Outside IP를 Terminal History나 문서에 출력하지 않습니다.

## 10. Cloud Route 구성

---

Cloud Workload Subnet의 Route Table에서 Virtual Private Gateway Route Propagation을 활성화합니다. VPN Connection이 `UP`이면 Static VPN Prefix인 `10.10.0.0/16` Route가 전파됩니다.

| Destination | Target | Origin |
| --- | --- | --- |
| `10.20.0.0/16` | `local` | Create Route Table |
| `10.10.0.0/16` | Virtual Private Gateway | Propagated |

Static Route를 직접 추가하는 경우에도 Destination은 On-Premise CIDR이고 Target은 Virtual Private Gateway입니다. 동일 Destination의 Static Route와 Propagated Route가 동시에 있으면 Route 우선순위를 확인합니다.

## 11. 연결 검증

---

검증 순서는 다음과 같습니다.

1. `[AWS-CONSOLE]`에서 두 VPN Tunnel의 상태를 확인합니다.

2. `[ROUTER-EC2]`에서 `sudo ipsec statusall`로 IKE와 Child SA를 확인합니다.

3. `[ONPREM-CLIENT]`의 Route가 `10.20.0.0/16`을 Router EC2로 보내는지 확인합니다.

4. `[CLOUD-TARGET]` Security Group이 `10.10.0.0/16`의 Test Traffic을 허용하는지 확인합니다.

5. On-Premise Client에서 Cloud Target Private IP로 Test합니다.

```bash
# [ONPREM-CLIENT]
ip route get <CLOUD_TARGET_PRIVATE_IP>
ping -c 3 <CLOUD_TARGET_PRIVATE_IP>
nc -vz <CLOUD_TARGET_PRIVATE_IP> <APP_PORT>
```

통신이 실패하면 Tunnel 상태만 보지 말고 다음 왕복 경로를 확인합니다.

```text
On-Premise Client Route
→ Router EC2 IP Forwarding
→ StrongSwan Tunnel
→ Cloud Route Table
→ Cloud Security Group·Network ACL
→ Return Route
```

## 12. 고가용성과 운영 설계

---

- AWS가 제공하는 두 Tunnel을 모두 구성하고 Failover를 Test합니다.

- On-Premise 단일 Router가 장애점이 되지 않도록 두 Customer Gateway Device와 별도 전원·회선을 검토합니다.

- BGP 지원 Device는 Dynamic Routing을 사용해 경로 변경과 Tunnel 장애 감지를 자동화합니다.

- Direct Connect를 Primary 경로로 사용할 때도 Site-to-Site VPN을 Backup 또는 암호화 계층으로 사용할 수 있습니다.

- MTU와 TCP MSS가 Tunnel Overhead를 고려하는지 확인합니다.

- CloudWatch Tunnel Metric과 VPN Log를 이용해 상태와 협상 Error를 감시합니다.

## 13. Resource 정리

---

실습 후 다음 순서로 제거합니다.

1. Site-to-Site VPN Connection을 삭제합니다.

2. Customer Gateway Resource를 삭제합니다.

3. Virtual Private Gateway를 Cloud VPC에서 Detach한 뒤 삭제합니다.

4. Router와 Client EC2를 종료하고 Elastic IP를 Release합니다.

5. Route, Security Group, Subnet, Internet Gateway와 두 VPC를 삭제합니다.

VPN Connection을 삭제하면 기존 Pre-Shared Key도 사용할 수 없게 됩니다. 실제 Key가 문서나 Log에 노출됐다면 연결을 유지한 채 숨기는 것으로 끝내지 말고 Tunnel Option의 Key를 교체합니다.

> **최종 정리**
> - Site-to-Site VPN은 Customer Gateway와 AWS Gateway 사이에 두 IPsec Tunnel을 제공합니다.
>
> - BGP Dynamic Routing은 경로 광고와 Tunnel Failover 판단을 자동화하고 Static Routing은 Prefix를 수동 관리합니다.
>
> - VPN으로 연결할 Network CIDR는 서로 겹치면 안 됩니다.
>
> - EC2 Router Lab에서는 Source/Destination Check와 Kernel IP Forwarding을 함께 설정합니다.
>
> - Pre-Shared Key와 Tunnel Configuration은 Secret이므로 Repository와 게시물에 저장하지 않습니다.

## 참고 자료

---

- [AWS Site-to-Site VPN 시작](https://docs.aws.amazon.com/vpn/latest/s2svpn/SetUpVPNConnections.html)

- [Static Routing과 Dynamic Routing](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-static-dynamic.html)

- [VPN Customer Gateway Device Configuration](https://docs.aws.amazon.com/vpn/latest/s2svpn/cgw-static-routing-examples.html)

- [Virtual Private Gateway Route Propagation](https://docs.aws.amazon.com/vpc/latest/userguide/WorkWithRouteTables.html#route-table-propagation)

- [VPC Route 우선순위](https://docs.aws.amazon.com/vpc/latest/userguide/route-tables-priority.html)
