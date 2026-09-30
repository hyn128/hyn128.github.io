---
title: EC2 Database 구축과 Amazon RDS
description: EC2에 MySQL과 MongoDB를 직접 설치하고 외부 연결을 제한적으로 구성한 뒤 AWS Database 종류와 Amazon RDS의 Managed Service 구조를 비교합니다
date: 2026-09-29
updated_at: 2026-09-30
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> Database를 EC2에 직접 설치하면 OS부터 Database Engine까지 사용자가 관리합니다. Amazon RDS는 지원되는 관계형 Database Engine의 Backup, Monitoring과 Maintenance 작업을 AWS가 관리합니다. 이 글에서는 EC2에 MySQL과 MongoDB를 설치하고 Network 접근을 제한하는 방법을 먼저 살펴본 뒤, AWS Database 종류와 RDS 설정 항목을 정리합니다.

## 1. EC2 Database와 Managed Database

---

EC2에 Database를 설치하는 방식과 Amazon RDS를 사용하는 방식은 관리 책임이 다릅니다.

| 항목 | EC2에 직접 설치 | Amazon RDS |
| --- | --- | --- |
| OS 관리 | 사용자가 Patch와 설정 관리 | AWS가 기반 OS 관리 |
| Database 설치 | 사용자가 설치·Upgrade | 지원 Engine과 Version을 선택해 생성 |
| Backup | 사용자가 도구와 보존 정책 구성 | Automated Backup과 Snapshot 제공 |
| 고가용성 | Replication과 Failover 직접 구성 | Multi-AZ 배포 Option 제공 |
| 접근 권한 | OS, Firewall, Database 계정 모두 관리 | VPC, Security Group과 Database 계정 관리 |
| 자유도 | Engine, Extension과 OS를 폭넓게 선택 | 지원되는 Engine, Version과 Option 범위 내에서 사용 |

특정 Version이나 OS 수준의 설정이 필요하면 EC2 방식이 적합할 수 있습니다. 일반적인 관계형 Database 운영에서 OS 관리 부담을 줄이려면 RDS를 먼저 검토합니다.

## 2. Database Network 구성 원칙

---

MySQL의 TCP 3306과 MongoDB의 TCP 27017을 `0.0.0.0/0`에 공개하지 않습니다. Database Process가 모든 Interface에서 Listen하도록 설정하더라도 Security Group은 신뢰할 수 있는 Source만 허용해야 합니다.

권장 흐름은 다음과 같습니다.

```text
외부 관리자
    ↓ VPN, SSM Port Forwarding 또는 SSH Tunnel
관리용 EC2
    ↓ Private IP
Database EC2
```

Application Server가 Database에 연결할 때는 IP CIDR보다 Application Security Group을 Source로 지정합니다.

| Database | Port | 권장 Source |
| --- | ---: | --- |
| MySQL | TCP 3306 | Application EC2 Security Group |
| MongoDB | TCP 27017 | Application EC2 Security Group |
| 실습 Client | 해당 Database Port | 현재 Client Public IP `/32`, 실습 후 제거 |

외부 접속 실습에서는 Public IP를 가진 Database Server를 그대로 운영 환경으로 사용하지 않습니다. 실습이 끝나면 Inbound Rule을 제거하고 Database를 Private Subnet으로 이동하거나 Private 연결 방식으로 전환합니다.

## 3. EC2에 MySQL 설치

---

다음 명령은 Ubuntu EC2 Instance에서 실행합니다.

### 3.1 설치와 상태 확인

```bash
sudo apt update
sudo apt install -y mysql-server
mysql --version
sudo systemctl enable --now mysql
systemctl is-active mysql
sudo ss -lntp | grep mysqld
```

Ubuntu Package로 설치한 MySQL은 Local 관리 계정에 Socket 인증을 사용할 수 있습니다. 먼저 `sudo mysql`로 접속해 실제 Account와 인증 방식을 확인합니다.

```bash
sudo mysql
```

MySQL 8.4에서 `mysql_native_password`는 기본적으로 비활성화되어 있고 MySQL 9.0에서는 제거되었습니다. Root Account를 `mysql_native_password`로 변경하지 않고 Application 전용 Account를 생성합니다.

### 3.2 Database와 전용 사용자 생성

MySQL Shell에서 실행합니다.

```sql
CREATE DATABASE appdb
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

CREATE USER 'appuser'@'10.0.%'
  IDENTIFIED BY '<STRONG_PASSWORD>';

GRANT ALL PRIVILEGES ON appdb.*
  TO 'appuser'@'10.0.%';

SHOW GRANTS FOR 'appuser'@'10.0.%';
```

`10.0.%`는 예제입니다. 실제 VPC CIDR과 Application 위치에 맞춰 Host 범위를 더 좁게 지정합니다. `'appuser'@'%'`처럼 모든 Host를 허용하는 Account는 실습에서도 피합니다. `CREATE USER`와 `GRANT`는 권한 Table에 즉시 반영되므로 이 절차에는 `FLUSH PRIVILEGES`가 필요하지 않습니다.

### 3.3 Network Listen 주소 변경

`/etc/mysql/mysql.conf.d/mysqld.cnf`의 `bind-address`를 EC2의 Private IP로 설정합니다.

```ini
[mysqld]
bind-address = <EC2_PRIVATE_IP>
```

설정을 반영하고 Listen 주소를 확인합니다.

```bash
sudo systemctl restart mysql
systemctl is-active mysql
sudo ss -lntp | grep 3306
```

`bind-address = 0.0.0.0`은 모든 IPv4 Interface에서 요청을 받습니다. 여러 Interface가 있는 Server에서는 Database 전용 Private IP를 지정하는 편이 안전합니다.

### 3.4 Security Group과 연결 확인

Database EC2 Security Group에 TCP 3306 Inbound Rule을 추가합니다. Source는 Application EC2 Security Group 또는 제한된 실습 Client `/32`로 지정합니다.

허용된 Client에서 연결합니다.

```bash
mysql \
  --host=<MYSQL_PRIVATE_IP> \
  --port=3306 \
  --user=appuser \
  --password \
  appdb
```

연결되지 않으면 MySQL Process, Listen 주소, EC2 Security Group, Network ACL과 Client Route 순서로 확인합니다.

## 4. EC2에 MongoDB 설치

---

MongoDB Package Repository는 Version과 Ubuntu Release에 따라 경로가 달라집니다. 다음 예제는 MongoDB Community Edition 8.0과 Ubuntu 24.04 LTS인 `noble`을 사용합니다. Ubuntu 22.04 LTS는 공식 문서에서 `jammy`용 Repository를 선택합니다.

### 4.1 Ubuntu Release 확인

```bash
cat /etc/os-release
dpkg --print-architecture
```

MongoDB 8.0 Community Edition이 지원하는 Ubuntu LTS Version과 Architecture인지 먼저 확인합니다.

### 4.2 공식 Repository 등록과 설치

```bash
sudo apt-get update
sudo apt-get install -y gnupg curl

curl -fsSL https://pgp.mongodb.com/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg \
  --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt-get update
sudo apt-get install -y mongodb-org
sudo systemctl enable --now mongod
systemctl is-active mongod
```

Local Shell 접속을 확인합니다.

```bash
mongosh
```

### 4.3 관리자 계정 생성

MongoDB는 처음에 Localhost에서 접속해 관리자 Account를 생성한 뒤 인증을 활성화합니다.

```javascript
use admin

db.createUser({
  user: "adminuser",
  pwd: passwordPrompt(),
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})
```

Blog 예제에 실제 Password를 기록하지 않습니다. 운영 환경에서는 업무별 Database와 최소 권한 Role을 가진 Account를 별도로 생성합니다.

### 4.4 인증과 Private Network 연결 활성화

`/etc/mongod.conf`에서 Localhost와 EC2 Private IP만 Listen하고 인증을 사용하도록 설정합니다.

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1,<EC2_PRIVATE_IP>

security:
  authorization: enabled
```

설정을 검증하기 위해 Service를 재시작하고 Log를 확인합니다.

```bash
sudo systemctl restart mongod
systemctl is-active mongod
sudo ss -lntp | grep 27017
sudo journalctl -u mongod --no-pager -n 50
```

MongoDB 공식 문서는 여러 Network Interface가 있을 때 Private 또는 내부 Interface에 Bind하도록 권고합니다. `bindIp: 0.0.0.0`과 인증 비활성화를 함께 사용하면 Internet에서 Database를 직접 조작할 수 있으므로 적용하지 않습니다.

MongoDB EC2 Security Group에는 TCP 27017을 Application Security Group 또는 실습 Client `/32`에만 허용합니다. 허용된 Client에서 다음과 같이 접속합니다.

```bash
mongosh \
  "mongodb://adminuser@<MONGODB_PRIVATE_IP>:27017/admin" \
  --password
```

### 4.5 MongoDB 제거

다음 명령은 MongoDB Package와 Data를 제거합니다. `/var/lib/mongodb`를 삭제하면 Database Data를 복구할 수 없으므로 Snapshot이나 Backup이 필요한지 먼저 확인합니다.

```bash
sudo systemctl stop mongod
sudo apt-get purge 'mongodb-org*'
sudo rm -r /var/log/mongodb
sudo rm -r /var/lib/mongodb
```

## 5. AWS Database Service 종류

---

Data Model과 Access Pattern에 따라 적합한 Database Service가 달라집니다.

| 유형 | 주요 용도 | AWS Service |
| --- | --- | --- |
| 관계형 Database | Transaction, ERP, CRM과 전자상거래 | Amazon RDS, Amazon Aurora |
| Data Warehouse | 대규모 분석과 집계 | Amazon Redshift |
| Key-Value | 대규모 Web, Game과 낮은 지연 시간 조회 | Amazon DynamoDB |
| In-memory | Cache, Session과 순위표 | Amazon ElastiCache, Amazon MemoryDB |
| Document | Content, Catalog와 User Profile | Amazon DocumentDB |
| Wide-column | 대규모 분산 Workload와 Apache Cassandra 호환 | Amazon Keyspaces |
| Graph | Fraud Detection, Social Network와 Recommendation | Amazon Neptune |
| Time Series | IoT, DevOps Metric과 산업 Telemetry | Amazon Timestream for LiveAnalytics |
| Ledger | 변경 이력을 검증하는 원장 | Amazon QLDB는 2025년 7월 31일 지원 종료 |

Amazon Redshift는 SQL을 사용하지만 Transaction 중심의 일반 RDBMS보다 분석용 Data Warehouse에 가깝습니다. Amazon Timestream for LiveAnalytics는 2025년 6월 20일부터 신규 고객에게 제공되지 않으며 기존 고객만 계속 사용할 수 있습니다.

Amazon DocumentDB는 MongoDB 호환 API를 제공하지만 MongoDB Server의 모든 기능과 동작이 같지는 않습니다. Migration 전에 사용하는 Driver, Operator와 Query의 호환성을 확인합니다.

## 6. Amazon RDS

---

Amazon RDS(Relational Database Service)는 관계형 Database Engine을 VPC 안에 배포하고 운영하는 Managed Service입니다. 사용자는 Schema, Query, Account와 Application Data를 관리하고 AWS는 기반 Infrastructure와 OS, Backup·Monitoring 기능을 관리합니다.

AWS Database Migration Service(AWS DMS)는 지원되는 Source와 Target 사이에서 Database를 Migration하거나 변경 Data를 지속적으로 복제할 때 사용합니다. Engine 조합에 따라 변환할 수 없는 Schema와 기능이 있으므로 AWS Schema Conversion Tool 또는 Engine별 변환 절차를 함께 검토합니다.

### 6.1 지원 Engine

RDS DB Instance에서는 다음 Engine을 사용할 수 있습니다.

- Amazon RDS for Db2입니다.

- MariaDB입니다.

- Microsoft SQL Server입니다.

- MySQL입니다.

- Oracle Database입니다.

- PostgreSQL입니다.

Amazon Aurora는 MySQL-Compatible Edition과 PostgreSQL-Compatible Edition을 제공합니다. Engine Version, Instance Class와 지원 Region의 조합은 달라질 수 있으므로 생성 전에 현재 지원 표를 확인합니다.

### 6.2 RDS가 관리하는 범위

| AWS가 관리하는 항목 | 사용자가 결정하는 항목 |
| --- | --- |
| 기반 Host와 OS | Database Engine과 Version |
| Hardware 교체 | DB Instance Class |
| Automated Backup 기능 | Backup 보존 기간과 Window |
| Monitoring 기능 제공 | Alarm, Log Export와 대응 절차 |
| Maintenance Update 제공 | Maintenance Window와 적용 시점 |
| Multi-AZ 기능 제공 | 배포 방식과 비용 수준 |

Managed Service라고 해서 모든 Update가 임의의 시점에 즉시 적용되는 것은 아닙니다. Maintenance Action의 적용 가능 시점과 자동 적용 여부를 확인하고 Preferred Maintenance Window를 업무 시간과 겹치지 않게 설정합니다. 긴급 보안 Update나 지원 종료 Version은 연기를 계속할 수 없습니다.

## 7. DB Instance Class와 비용

---

DB Instance Class는 EC2 Instance Type과 비슷하게 vCPU, Memory, Network와 Storage 처리 성능을 결정합니다.

| Class 계열 예 | 목적 |
| --- | --- |
| `db.t` | 개발·실습과 Burstable Workload |
| `db.m` | 범용 Workload |
| `db.r`, `db.x` | Memory 중심 Workload |

모든 Engine과 Version이 모든 Class를 지원하는 것은 아닙니다. Console 또는 공식 지원 표에서 선택한 Region, Engine과 Version 조합을 확인합니다.

RDS 비용에는 다음 항목이 포함될 수 있습니다.

- DB Instance 실행 시간 또는 Reserved Instance 비용입니다.

- Provisioning한 Storage, IOPS와 Throughput 비용입니다.

- Multi-AZ와 Read Replica에 추가로 생성되는 Resource 비용입니다.

- Backup 보존량과 Manual Snapshot 비용입니다.

- License가 필요한 상용 Engine의 License 비용입니다.

- AZ, Region과 Internet 경로에 따른 Data Transfer 비용입니다.

Backup Storage 무료 범위를 단순히 `Database Storage의 100%`로만 계산하지 않습니다. Region의 활성 RDS Resource와 Backup 유형에 따라 과금 조건이 달라질 수 있으므로 현재 RDS Pricing과 Billing 내역을 확인합니다.

## 8. Amazon Aurora

---

Amazon Aurora는 AWS가 Cloud 환경에 맞게 설계한 관계형 Database이며 MySQL 또는 PostgreSQL 호환 Edition을 제공합니다.

Aurora는 Compute Instance와 분산 Storage 계층을 분리합니다. Writer와 Reader가 같은 Cluster Storage를 사용하므로 일반적인 EC2 기반 MySQL Replication과 Data 전달 방식이 다릅니다.

호환 Edition이라는 표현이 모든 Extension, Version과 동작이 동일하다는 뜻은 아닙니다. Migration 전에 SQL, Character Set, Plugin, Stored Procedure와 Driver 호환성을 확인합니다.

다음 기준으로 배포 방식을 선택합니다.

| 방식 | 선택 기준 |
| --- | --- |
| EC2에 MySQL·PostgreSQL 설치 | OS, Extension과 Engine 설정을 직접 제어해야 하는 경우 |
| Amazon RDS | 일반적인 관계형 Database 운영과 관리 부담 감소가 필요한 경우 |
| Amazon Aurora | Aurora의 분산 Storage, Read Scaling과 Cluster 기능이 필요한 경우 |

## 9. RDS 설정 항목

---

RDS DB Instance를 생성할 때 다음 항목을 함께 결정합니다.

| 영역 | 주요 설정 | 확인 사항 |
| --- | --- | --- |
| Engine | Database 종류와 Version | Application 호환성과 지원 종료 일정 |
| Instance | DB Instance Class | vCPU, Memory, Network와 예상 비용 |
| Availability | Single-AZ, Multi-AZ | 장애 허용 수준과 비용 |
| Storage | gp3, io1·io2와 할당량 | IOPS, Throughput, 암호화와 Auto Scaling |
| Identifier | DB Instance Identifier | Account와 Region 안에서 구분 가능한 이름 |
| Credential | Master User와 인증 방식 | Secrets Manager 또는 안전한 비밀 관리 |
| Network | VPC, DB Subnet Group, Public Access | EC2와의 Private 연결 경로 |
| Security | VPC Security Group, KMS | 허용 Source와 암호화 Key |
| Database 환경 | Database 이름, Port, Parameter Group, Option Group | Engine별 Parameter와 변경 적용 시점 |
| Backup | 보존 기간과 Backup Window | 복구 목표와 Maintenance Window 중복 여부 |
| Monitoring | Enhanced Monitoring, Performance Insights, Log Export | 관찰할 Metric과 보존 비용 |
| Maintenance | Preferred Maintenance Window | Application 점검 시간과 Upgrade 계획 |
| Protection | Deletion Protection, Final Snapshot | 오삭제 방지와 종료 절차 |

`Public access 가능`은 다른 VPC에서 자동으로 접근할 수 있게 만드는 Option이 아닙니다. 활성화하면 외부에서 Endpoint를 조회할 때 Public IP로 해석될 수 있으며, 실제 접속은 Internet Gateway 경로와 Security Group Rule도 충족해야 합니다.

일반적인 Application 구성에서는 RDS를 Private Subnet에 두고 Application EC2 Security Group만 Database Port에 접근하도록 허용합니다.

## 10. RDS for MySQL 생성

---

`[RDS] → [데이터베이스] → [데이터베이스 생성]`에서 다음 순서로 진행합니다.

1. 생성 방식을 선택하고 Engine으로 MySQL을 선택합니다.

2. Application이 지원하는 MySQL Version을 선택합니다.

3. 실습 또는 운영 목적에 맞는 Template을 선택합니다.

4. DB Instance Identifier와 Master User를 지정합니다. Password는 문서나 Source Code에 기록하지 않습니다.

5. DB Instance Class와 Storage 유형·용량을 선택합니다.

6. 고가용성이 필요하면 Multi-AZ 배포를 선택합니다. 실습 비용을 줄이기 위한 Single-AZ와 운영 고가용성 구성을 구분합니다.

7. Application EC2와 연결 가능한 VPC와 DB Subnet Group을 선택합니다.

8. Public Access는 기본적으로 `아니요`를 선택합니다.

9. RDS Security Group의 TCP 3306 Source로 Application EC2 Security Group을 지정합니다.

10. Backup 보존 기간, Backup Window, Maintenance Window, Log Export와 Deletion Protection을 확인합니다.

11. 예상 월간 비용을 확인하고 DB Instance를 생성합니다.

DB Instance 상태가 `Available`이 되면 Endpoint와 Port를 확인합니다. Application EC2에서 다음과 같이 연결합니다.

```bash
mysql \
  --host=<RDS_ENDPOINT> \
  --port=3306 \
  --user=<MASTER_OR_APP_USER> \
  --password
```

접속이 실패하면 다음 순서로 확인합니다.

1. RDS 상태가 `Available`인지 확인합니다.

2. EC2와 RDS의 VPC, Subnet Route와 DNS Resolution을 확인합니다.

3. RDS Security Group이 EC2 Security Group을 Source로 허용하는지 확인합니다.

4. Endpoint와 Port가 올바른지 확인합니다.

5. Database Account의 Host 범위와 권한을 확인합니다.

6. TLS가 필요한 Client Option과 Certificate 구성을 확인합니다.

## 11. EC2와 RDS의 실행 흐름

---

```text
Client
  ↓ HTTP/HTTPS
Application Load Balancer
  ↓
Application EC2
  ↓ TCP 3306, RDS Security Group
Private RDS for MySQL
  ├── Automated Backup
  ├── Monitoring
  └── Multi-AZ Option
```

Client는 Database에 직접 접속하지 않습니다. Application EC2가 Business Logic을 처리한 뒤 Private Endpoint로 RDS에 Query를 전송합니다. RDS Security Group은 Application EC2 Security Group에서 시작된 연결만 허용합니다.

## 12. 실습 후 정리

---

다음 Resource는 Instance를 Stop하는 것만으로 비용이 모두 중지되지 않습니다.

- 필요하지 않은 RDS DB Instance를 삭제하고 Final Snapshot 보존 여부를 결정합니다.

- Manual Snapshot과 Automated Backup 보존 상태를 확인합니다.

- EC2 Database Instance와 연결된 EBS Volume, Snapshot과 Elastic IP를 확인합니다.

- 실습용 TCP 3306과 27017 Inbound Rule을 제거합니다.

- 더 이상 사용하지 않는 DB Subnet Group, Parameter Group과 Security Group의 참조 여부를 확인합니다.

RDS를 삭제할 때 Deletion Protection이 활성화되어 있으면 먼저 비활성화해야 합니다. Final Snapshot을 생략하면 삭제 시점의 Database 상태로 복구할 수 없습니다.

다음 글인 [DynamoDB·ElastiCache와 Amazon DocumentDB](/cloud-08-aws-dynamodb-elasticache-documentdb/)에서는 Key-Value·In-memory Database의 용도와 MongoDB 호환 Document Database의 연결 구조를 살펴봅니다.

## 참고 자료

---

- [Ubuntu MySQL Server 설치와 설정](https://ubuntu.com/server/docs/install-and-configure-a-mysql-server)

- [MySQL Native Password 상태](https://dev.mysql.com/doc/refman/8.4/en/native-pluggable-authentication.html)

- [MongoDB Community Edition 8.0 Ubuntu 설치](https://www.mongodb.com/docs/v8.0/tutorial/install-mongodb-on-ubuntu/)

- [MongoDB IP Binding 보안](https://www.mongodb.com/docs/manual/core/security-mongodb-configuration/)

- [Amazon RDS 개요](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/)

- [VPC의 RDS DB Instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html)

- [Amazon RDS DB Instance 생성](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateDBInstance.html)

- [지원 종료된 AWS Service](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html)
