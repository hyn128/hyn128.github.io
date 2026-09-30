---
title: DynamoDB·ElastiCache와 Amazon DocumentDB
description: AWS의 Key-Value·In-memory Database인 DynamoDB와 ElastiCache를 비교하고 Amazon DocumentDB Cluster를 Private Network에서 안전하게 연결하는 방법을 정리합니다
date: 2026-09-30
series: Cloud
tags:
  - Cloud
  - AWS
  - AutoEverSW
---

## 요약

---

> DynamoDB는 Key를 기준으로 Item을 조회하는 Serverless NoSQL Database이고, ElastiCache는 반복 조회 결과와 Session처럼 빠르게 접근해야 하는 데이터를 Memory에 보관하는 Managed Cache입니다. Amazon DocumentDB는 MongoDB 호환 API를 제공하는 Managed Document Database이며 VPC 내부에서 접근하는 것이 기본입니다. 이 글에서는 세 Service의 역할을 구분하고 DocumentDB를 EC2와 Application에서 연결하는 흐름을 정리합니다.

## 1. Data Model에 따른 Database 선택

---

관계형 Database가 Table 사이의 관계와 Transaction을 중심으로 Data를 다룬다면 Key-Value Database는 고유한 Key로 Value를 빠르게 찾는 Access Pattern에 집중합니다.

```text
Key                         Value
user:1001        →          {name: "adam", role: "engineer"}
session:8f2a     →          {userId: 1001, expiresAt: ...}
```

Schema를 먼저 고정하고 Join하는 방식보다 조회에 사용할 Key와 예상 Query를 먼저 설계합니다. 대규모 Web Service, Game State와 IoT Data처럼 요청량이 많고 짧은 응답 시간이 필요한 경우에 적합합니다.

| 구분 | DynamoDB | ElastiCache | DocumentDB |
| --- | --- | --- | --- |
| 주된 역할 | 지속 가능한 Key-Value·Document Data 저장 | Cache, Session, Ranking | MongoDB 호환 Document Data 저장 |
| 저장 위치 | AWS가 관리하는 분산 Storage | Memory 중심 | AWS가 관리하는 분산 Cluster Storage |
| 접근 방식 | HTTPS API와 AWS SDK | Valkey·Redis OSS·Memcached Protocol | MongoDB Driver와 Wire Protocol |
| Network | Public Service Endpoint 또는 VPC Endpoint | VPC 내부 | VPC 내부 |
| 대표 선택 기준 | Key 기반 조회와 자동 확장 | 매우 짧은 지연 시간과 반복 조회 감소 | JSON 형태의 Document와 MongoDB 호환 API |

## 2. Amazon DynamoDB

---

DynamoDB는 Server를 직접 생성하거나 Patch하지 않고 Table을 만드는 Serverless NoSQL Database입니다. Item은 Primary Key로 식별하며 Key-Value와 Document Data Model을 지원합니다.

주요 특성은 다음과 같습니다.

- AWS SDK가 DynamoDB HTTPS API를 호출하므로 RDBMS처럼 Database Process에 TCP Session을 맺고 Connection Pool을 관리하지 않습니다. 다만 SDK 내부의 HTTP Connection 재사용과 Timeout 설정은 여전히 성능에 영향을 줍니다.

- 단일 Item 작업 외에 ACID Transaction을 지원합니다.

- Encryption at Rest, IAM 기반 접근 제어와 VPC Endpoint를 사용할 수 있습니다.

- Global Table을 사용하면 여러 Region에 Replica를 두고 변경 사항을 복제할 수 있습니다.

- Capacity Mode는 On-demand와 Provisioned 중에서 선택합니다.

| Capacity Mode | 동작 | 적합한 경우 |
| --- | --- | --- |
| On-demand | 실제 Read·Write 요청량을 기준으로 과금 | 요청량을 예측하기 어렵거나 급변하는 Workload |
| Provisioned | RCU와 WCU를 지정하고 필요하면 Auto Scaling 적용 | 요청량이 예측 가능하고 Capacity를 직접 관리할 Workload |

Serverless라는 표현이 언제나 `요청이 없으면 비용이 전혀 없다`는 뜻은 아닙니다. Provisioned Capacity, Backup, Global Table, Data Transfer와 Storage 사용량에 따라 비용이 발생합니다.

## 3. Amazon ElastiCache

---

ElastiCache는 Valkey, Redis OSS와 Memcached Engine을 지원하는 Managed In-memory Cache입니다. Application과 원본 Database 사이에 Cache를 두면 반복 Query와 원본 Database 부하를 줄일 수 있습니다.

```text
Application
  ├── Cache Hit  → ElastiCache에서 즉시 응답
  └── Cache Miss → Database 조회 → Cache 저장 → 응답
```

| Engine | 주요 특징 | 대표 용도 |
| --- | --- | --- |
| Valkey·Redis OSS | 다양한 Data Type, Replication과 Persistence Option 지원 | Session, Ranking, Cache |
| Memcached | 단순한 분산 Key-Value Cache와 Multi-thread 처리 | 일시적인 Object Cache |

In-memory Service를 모두 `재시작하면 Data가 사라지는 Database`로 설명할 수는 없습니다. Valkey와 Redis OSS는 구성에 따라 Replica와 Backup을 사용할 수 있습니다. 그러나 Cache Data는 제거되거나 만료될 수 있다는 전제로 설계하고, 복구할 수 없는 원본 Data의 유일한 저장소로 사용하지 않습니다.

## 4. Amazon DocumentDB 구조

---

> Amazon DocumentDB는 MongoDB 호환 API를 제공하는 Managed Document Database입니다. MongoDB 자체를 Hosting하는 Service가 아니므로 지원 Command, Index와 동작 차이를 Migration 전에 확인해야 합니다.

Document는 BSON 형태로 저장되며 Field가 다른 Data를 같은 Collection에서 다룰 수 있습니다. Cluster는 Instance와 Cluster Storage로 구성되고, Application은 Cluster Endpoint 또는 Reader Endpoint에 연결합니다.

```text
Application EC2
  ↓ TCP 27017, TLS
DocumentDB Cluster Endpoint
  ├── Writer Instance
  ├── Reader Instance
  └── Cluster Storage
```

DocumentDB Cluster는 VPC에 배포됩니다. Internet에서 직접 접근할 Public Endpoint를 제공하지 않으므로 Application은 같은 VPC, 연결된 VPC 또는 Private Network 경로에서 접속해야 합니다.

## 5. DocumentDB Cluster 생성

---

`[Amazon DocumentDB] → [클러스터] → [생성]`에서 다음 항목을 결정합니다.

1. Application과 Driver가 지원하는 Engine Version을 선택합니다.

2. 예상 Data와 Query 부하에 맞는 Instance Class를 선택합니다.

3. Application EC2와 통신할 수 있는 VPC와 DB Subnet Group을 선택합니다.

4. 운영 고가용성이 필요하면 여러 Availability Zone에 Instance를 배치합니다.

5. Master Username을 지정하고 Password는 AWS Secrets Manager 또는 별도 비밀 관리 수단에 저장합니다.

6. DocumentDB Security Group의 TCP 27017 Source로 Application EC2 Security Group을 지정합니다.

7. Backup 보존 기간, Maintenance Window, Encryption과 Log Export를 확인합니다.

EC2 연결 Option은 필요한 Network Rule을 구성하는 데 도움을 줄 수 있지만, 선택하지 않았다고 DocumentDB가 VPC 밖에 생성되는 것은 아닙니다. 접속 가능 여부는 VPC Route, Security Group과 DNS Resolution이 함께 결정합니다.

## 6. EC2에서 DocumentDB 연결

---

다음 명령은 DocumentDB와 같은 VPC 또는 연결된 Private Network의 EC2에서 실행합니다.

### 6.1 CA Bundle 다운로드

TLS Server Certificate를 검증하기 위해 AWS Trust Store의 CA Bundle을 받습니다.

```bash
curl -fsSLo global-bundle.pem \
  https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

파일이 내려받아졌는지 확인합니다.

```bash
test -s global-bundle.pem
openssl x509 -in global-bundle.pem -noout -subject 2>/dev/null || true
```

Bundle에는 여러 Certificate가 들어 있으므로 첫 Certificate 정보만 출력되거나 `openssl x509` 확인이 제한될 수 있습니다. Application 연결 시 이 File을 CA File로 지정합니다.

### 6.2 mongosh 연결

Password를 명령행 인수로 직접 남기지 않고 Prompt에서 입력합니다.

```bash
mongosh \
  --host <DOCUMENTDB_CLUSTER_ENDPOINT> \
  --port 27017 \
  --tls \
  --tlsCAFile global-bundle.pem \
  --retryWrites=false \
  --username <DOCUMENTDB_USER> \
  --password
```

DocumentDB는 Retryable Write를 지원하지 않으므로 `retryWrites=false`를 사용합니다. 연결 후 `ping`으로 통신을 확인합니다.

```javascript
db.runCommand({ ping: 1 })
```

## 7. 외부 Client의 SSH Port Forwarding

---

Local PC에서 일시적으로 관리해야 한다면 같은 VPC의 EC2를 Jump Host로 사용해 Local Port를 전달할 수 있습니다.

```text
Local PC localhost:27017
  ↓ SSH Local Forwarding, TCP 22
Jump Host EC2
  ↓ Private Network, TCP 27017
DocumentDB Cluster
```

### 7.1 Security Group

| 대상 | Inbound Source | Port |
| --- | --- | ---: |
| Jump Host EC2 | 관리자 Public IP `/32` | TCP 22 |
| DocumentDB | Jump Host Security Group | TCP 27017 |

DocumentDB의 TCP 27017을 `0.0.0.0/0`에 개방하지 않습니다.

### 7.2 Tunnel 생성

다음 명령은 Local PC에서 실행합니다.

```bash
ssh -i <EC2_KEY_PATH> \
  -L 27017:<DOCUMENTDB_CLUSTER_ENDPOINT>:27017 \
  <EC2_USER>@<EC2_PUBLIC_DNS> \
  -N
```

| Option | 역할 |
| --- | --- |
| `-i` | Jump Host 인증에 사용할 Private Key 지정 |
| `-L` | Local Port를 Remote Destination으로 전달 |
| `-N` | Remote Shell 없이 Port Forwarding만 수행 |

별도 Terminal에서 `localhost:27017`의 Listen 상태를 확인합니다.

```bash
lsof -nP -iTCP:27017 -sTCP:LISTEN
```

SSH Tunnel에서는 Client가 `localhost`에 연결하지만 Certificate는 DocumentDB Endpoint용이므로 Hostname 검증이 일치하지 않습니다. AWS의 외부 VPC 연결 예제는 이 경우 Invalid Hostname 허용 Option을 사용합니다. 이 Option은 Server Identity 검증을 약화하므로 임시 관리 실습에만 사용하고, 운영 Application은 VPC 내부에서 실제 Cluster Endpoint와 CA Bundle로 연결합니다.

Compass에서도 Host를 `localhost`, Port를 `27017`로 지정하고 Username, Password와 `global-bundle.pem`을 설정합니다. SSH 설정에는 Jump Host 주소, OS User와 Private Key를 지정합니다.

## 8. Python Application 연결

---

운영 Application은 DocumentDB와 Private Network로 통신하는 EC2, Container 또는 Lambda에 배치합니다. 다음 예제는 같은 VPC의 Application Host에서 실행합니다.

```bash
python -m pip install pymongo
```

Credential과 Endpoint는 Source Code에 넣지 않고 Environment Variable 또는 Secrets Manager에서 주입합니다.

```bash
export DOCDB_HOST='<DOCUMENTDB_CLUSTER_ENDPOINT>'
export DOCDB_USER='<DOCUMENTDB_USER>'
export DOCDB_PASSWORD='<DOCUMENTDB_PASSWORD>'
export DOCDB_CA_FILE='/opt/app/certs/global-bundle.pem'
```

```python
import os

from pymongo import MongoClient


client = MongoClient(
    host=os.environ["DOCDB_HOST"],
    port=27017,
    username=os.environ["DOCDB_USER"],
    password=os.environ["DOCDB_PASSWORD"],
    authSource="admin",
    tls=True,
    tlsCAFile=os.environ["DOCDB_CA_FILE"],
    replicaSet="rs0",
    readPreference="secondaryPreferred",
    retryWrites=False,
)

try:
    client.admin.command("ping")

    users = client["test_database"]["users"]
    result = users.insert_one(
        {"name": "adam", "role": "engineer", "framework": "FastAPI"}
    )
    print(result.inserted_id)
    print(users.find_one({"_id": result.inserted_id}))
finally:
    client.close()
```

실제 Service에서는 Environment Variable에 장기 Password를 직접 저장하는 대신 Secrets Manager에서 실행 시점에 조회하고, Application IAM Role에는 해당 Secret과 필요한 KMS Key에 대한 최소 권한만 부여합니다.

## 9. 연결 실패 점검

---

| 증상 | 확인 항목 |
| --- | --- |
| Connection Timeout | VPC Route, TCP 27017 Security Group, Endpoint와 DNS |
| Authentication Failed | Username, Password와 `authSource=admin` |
| TLS 오류 | CA Bundle 경로, File 권한과 Cluster TLS 설정 |
| Retryable Write 오류 | URI 또는 Driver Option의 `retryWrites=false` |
| Tunnel은 열렸지만 접속 실패 | SSH Process, Local Port 충돌과 Jump Host에서 Cluster 접근 가능 여부 |
| Migration 후 Query 오류 | DocumentDB Version별 지원 API와 MongoDB 기능 차이 |

## 10. 실습 후 정리

---

- 사용하지 않는 DocumentDB Cluster와 Instance를 삭제하기 전에 Final Snapshot 필요 여부를 확인합니다.

- 실습용 Jump Host를 종료하거나 삭제하고 Elastic IP, EBS Volume과 Snapshot을 확인합니다.

- Jump Host의 TCP 22 Rule과 DocumentDB의 임시 TCP 27017 Rule을 제거합니다.

- Local PC에 저장한 Private Key, CA Bundle과 Shell History의 Credential 노출 여부를 확인합니다.

다음 글인 [Route 53과 ACM으로 Domain·HTTPS 구성](/cloud-09-aws-route53-acm-https/)에서는 Domain을 EC2 또는 Application Load Balancer에 연결하고 ACM Certificate로 HTTPS Listener를 구성합니다.

## 참고 자료

---

- [DynamoDB Capacity Mode](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/capacity-mode.html)

- [DynamoDB Global Table 동작](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html)

- [ElastiCache Engine 비교](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/SelectEngine.html)

- [Amazon DocumentDB와 MongoDB의 기능 차이](https://docs.aws.amazon.com/documentdb/latest/devguide/functional-differences.html)

- [VPC 외부에서 DocumentDB 연결](https://docs.aws.amazon.com/documentdb/latest/developerguide/connect-from-outside-a-vpc.html)
