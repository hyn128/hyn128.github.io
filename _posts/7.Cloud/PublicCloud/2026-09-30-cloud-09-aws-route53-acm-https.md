---
title: Route 53과 ACM으로 Domain·HTTPS 구성
description: Route 53 Hosted Zone의 DNS Record를 EC2와 Application Load Balancer에 연결하고 ACM Public Certificate와 HTTPS Listener를 구성합니다
date: 2026-09-30
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> Route 53은 Domain 등록, Authoritative DNS와 Health Check 기능을 제공하고, AWS Certificate Manager는 AWS Service에 연결할 TLS Certificate를 발급·관리합니다. 이 글에서는 Domain과 Hosted Zone의 관계, EC2·Application Load Balancer에 맞는 DNS Record, ACM DNS 검증과 HTTPS Listener의 요청 흐름을 정리합니다.

## 1. Domain과 DNS

---

IP Address는 Server 위치를 나타내지만 사용자가 기억하기 어렵고 Infrastructure 변경에 따라 바뀔 수 있습니다. DNS는 `app.example.com` 같은 Domain Name을 실제 접속 대상과 연결합니다.

```text
Browser
  ↓ app.example.com 조회
DNS Resolver
  ↓ Authoritative DNS 질의
Route 53 Hosted Zone
  ↓ A 또는 Alias Record 응답
EC2 Public IP 또는 ALB
```

Route 53의 기능은 다음 세 영역으로 구분할 수 있습니다.

| 기능 | 역할 |
| --- | --- |
| Domain Registration | Domain을 등록하고 등록자 정보를 관리 |
| DNS Service | Hosted Zone과 Record로 DNS Query에 응답 |
| Health Check·Traffic Flow | Endpoint 상태와 Routing Policy에 따라 응답 대상 선택 |

Domain은 Route 53이 아닌 Registrar에서 구매해도 됩니다. 외부에서 등록한 Domain을 Route 53에서 관리하려면 Public Hosted Zone을 만들고 Registrar의 Name Server를 Route 53이 제공한 NS Record로 변경합니다.

## 2. Hosted Zone과 Record

---

> Hosted Zone은 특정 Domain의 DNS Record를 관리하는 Container입니다. Public Hosted Zone은 Internet DNS Query에 응답하고 Private Hosted Zone은 연결한 VPC 내부에서만 사용합니다.

| Record Type | 저장 값 | 대표 용도 |
| --- | --- | --- |
| `A` | IPv4 Address | EC2 Elastic IP 또는 IPv4 대상 |
| `AAAA` | IPv6 Address | IPv6 대상 |
| `CNAME` | 다른 Domain Name | Subdomain을 다른 이름에 연결 |
| `MX` | Mail Server | E-mail 수신 Server 지정 |
| `TXT` | Text | Domain 소유권 검증, SPF와 기타 Metadata |
| Alias `A`·`AAAA` | 지원되는 AWS Resource | ALB, CloudFront와 S3 Website Endpoint 등에 연결 |

Alias Record는 Route 53의 확장 기능입니다. Zone Apex인 `example.com`도 ALB처럼 지원되는 AWS Resource에 연결할 수 있고, 대상의 IP가 바뀌어도 Record를 직접 수정할 필요가 없습니다.

## 3. EC2에 Domain 직접 연결

---

단일 EC2에 직접 연결할 때는 Public IPv4가 변경되지 않도록 Elastic IP 사용을 검토합니다.

`[Route 53] → [호스팅 영역] → [레코드 생성]`에서 다음 값을 설정합니다.

| 항목 | 예제 |
| --- | --- |
| Record Name | `app` |
| Record Type | `A` |
| Value | `<EC2_ELASTIC_IP>` |
| TTL | 변경 계획과 Cache 영향을 고려해 선택 |

DNS 응답을 확인합니다.

```bash
dig +short app.example.com A
```

HTTP 응답까지 확인합니다.

```bash
curl -I http://app.example.com
```

DNS Record가 올바르더라도 EC2 Security Group, Network ACL, Web Server Listen Port와 OS Firewall이 요청을 허용해야 접속할 수 있습니다.

## 4. Application Load Balancer에 Domain 연결

---

여러 EC2에 Traffic을 분산하거나 HTTPS를 Load Balancer에서 종료하려면 ALB를 사용합니다.

```text
Client
  ↓ DNS Query
Route 53 Alias A/AAAA
  ↓
Application Load Balancer
  ↓ Target Group Health Check
EC2 Instances
```

### 4.1 Target Group 생성

1. Target Type과 Application Port를 선택합니다.

2. Health Check Protocol, Port와 Path를 설정합니다.

3. 요청을 처리할 EC2 Instance를 등록합니다.

4. 모든 Target이 `healthy` 상태인지 확인합니다.

### 4.2 ALB 생성

Internet에서 접속하는 Service라면 Internet-facing ALB를 두 개 이상의 Availability Zone Subnet에 배치합니다.

| Security Group | Inbound Rule |
| --- | --- |
| ALB | Client에서 TCP 80·443 |
| Application EC2 | ALB Security Group에서 Application Port |

Application EC2의 HTTP Port를 Internet 전체에 공개하지 않고 ALB Security Group만 Source로 허용합니다.

### 4.3 Alias Record 생성

Hosted Zone에서 Alias `A` Record를 만들고 대상 AWS Resource로 ALB를 선택합니다. IPv6를 제공한다면 Alias `AAAA` Record도 생성합니다.

```bash
dig +short app.example.com A
curl -I http://app.example.com
```

ALB의 Address는 고정 IP 한 개가 아니므로 ALB DNS Name을 일반 `A` Record의 Value에 IP로 복사하지 않습니다.

## 5. Route 53 Routing Policy와 Health Check

---

Route 53은 Record 구성에 따라 다양한 Routing Policy를 제공합니다.

| Policy | 목적 |
| --- | --- |
| Simple | 하나의 Resource 또는 단순한 여러 Value 응답 |
| Weighted | 설정한 Weight 비율로 Traffic 분배 |
| Latency | 요청 위치에서 Latency가 낮은 Region 선택 |
| Failover | Primary 장애 시 Secondary Record 응답 |
| Geolocation | 사용자 위치에 따라 대상 선택 |
| Multivalue Answer | 여러 정상 Record 중 일부 IP 응답 |

Health Check를 생성하는 것만으로 Traffic이 자동으로 다른 Region으로 이동하지는 않습니다. 동일 Name과 Type의 여러 Record에 Failover·Weighted 같은 Routing Policy를 구성하고 Health 상태를 연결해야 합니다.

ALB Alias Record에서는 `Evaluate target health`를 사용해 ALB와 Target Group 상태를 평가할 수 있습니다. 정상 대상이 하나뿐인 구성은 해당 대상 장애 시 대체할 Record가 없으므로 Multi-Region Failover가 되지 않습니다.

## 6. TLS와 HTTPS

---

TLS Certificate는 Client가 접속한 Domain의 Server Identity를 확인하고 전송 Data를 암호화하는 데 사용합니다.

```text
Client
  ↓ TLS Handshake, Certificate 검증
ALB HTTPS Listener :443
  ↓ HTTP 또는 HTTPS
Target Group
  ↓
Application EC2
```

`HTTPS는 IP Address에 적용할 수 없다`고 일반화할 수는 없습니다. IP Address가 Subject Alternative Name에 포함된 Certificate도 존재합니다. 다만 ACM Public Certificate는 Domain Name에 대해 발급하므로 이 구성에서는 Domain이 필요합니다.

## 7. ACM Public Certificate 발급

---

`[AWS Certificate Manager] → [인증서 요청] → [퍼블릭 인증서 요청]`에서 진행합니다.

1. `app.example.com`처럼 실제 HTTPS에 사용할 FQDN을 입력합니다.

2. `*.example.com` Wildcard가 필요하면 Apex Domain인 `example.com`을 별도 이름으로 추가합니다. Wildcard는 한 단계 아래 Subdomain만 보호합니다.

3. Validation Method로 DNS Validation을 선택합니다.

4. Route 53에서 관리하는 Domain이면 `[Route 53에서 레코드 생성]`으로 Validation CNAME을 추가합니다.

5. Certificate 상태가 `Issued`로 바뀌는지 확인합니다.

DNS Validation CNAME을 삭제하면 자동 갱신에 실패할 수 있습니다. Certificate가 지원 AWS Service에 연결되어 사용 중이고 Validation Record가 유지되어야 Managed Renewal 대상이 됩니다.

Certificate Region도 확인해야 합니다.

- ALB에 연결하는 ACM Certificate는 ALB와 같은 Region에 있어야 합니다.

- CloudFront에 연결하는 Certificate는 `us-east-1`에 요청하거나 Import해야 합니다.

## 8. ALB HTTPS Listener 구성

---

ALB의 `[리스너 및 규칙]`에서 TCP 443이 아니라 `HTTPS:443` Listener를 추가합니다.

| 설정 | 값 |
| --- | --- |
| Protocol·Port | `HTTPS:443` |
| Default Action | Application Target Group으로 Forward |
| Certificate | ACM에서 발급한 Domain Certificate |
| Security Policy | Application 호환성을 만족하는 현재 TLS Policy |

HTTP 요청도 허용해야 한다면 `HTTP:80` Listener의 Default Action을 HTTPS로 Redirect합니다.

```text
HTTP :80
  ↓ 301 Redirect
HTTPS :443
  ↓ Forward
Target Group
```

ALB Security Group에는 TCP 443을 허용하고, TCP 80은 Redirect를 제공할 때만 허용합니다.

## 9. 검증

---

DNS부터 Certificate와 HTTP 응답 순서로 확인합니다.

```bash
dig +short app.example.com A
```

```bash
openssl s_client \
  -connect app.example.com:443 \
  -servername app.example.com \
  </dev/null
```

```bash
curl -I http://app.example.com
curl -I https://app.example.com
```

정상 구성에서는 HTTP 요청이 HTTPS로 Redirect되고 HTTPS 요청은 `2xx` 또는 Application이 의도한 응답을 반환합니다.

| 증상 | 확인 항목 |
| --- | --- |
| Domain이 해석되지 않음 | Registrar의 NS, Hosted Zone과 Record Name |
| ACM이 Pending Validation | Validation CNAME의 Name·Value와 DNS 전파 |
| Certificate 경고 | 접속 Domain과 Certificate 이름, Certificate Chain |
| ALB `503` | Target Group에 등록된 정상 Target 존재 여부 |
| ALB `502` | Application 응답, Port와 Protocol 불일치 |
| HTTPS Timeout | ALB Security Group TCP 443과 Listener |

## 10. 실습 후 정리

---

- 사용하지 않는 Alias와 A Record를 삭제합니다.

- ALB와 Target Group을 삭제하고 관련 Security Group을 확인합니다.

- 사용하지 않는 ACM Certificate는 다른 AWS Resource에서 참조하는지 확인한 뒤 삭제합니다.

- Route 53 Hosted Zone과 등록 Domain은 별도 과금 대상이므로 보존 여부를 확인합니다.

- Domain을 다른 DNS Provider로 옮길 때 Registrar의 NS 변경과 DNSSEC 설정을 함께 확인합니다.

다음 글인 [Amazon S3 Backup과 Spring Boot File Upload](/cloud-10-aws-s3-backup-spring-upload/)에서는 S3와 Storage Gateway를 이용한 Backup 구조와 Private Bucket에 File을 올리는 Application을 구성합니다.

## 참고 자료

---

- [Route 53을 ALB에 연결](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html)

- [Route 53 Routing Policy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)

- [Route 53 Health Check와 Record 선택](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-how-route-53-chooses-records.html)

- [ACM Public Certificate 특징](https://docs.aws.amazon.com/acm/latest/userguide/acm-certificate-characteristics.html)

- [ACM Managed Renewal 조건](https://docs.aws.amazon.com/acm/latest/userguide/check-certificate-renewal-status.html)

- [ALB HTTPS Listener](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html)
