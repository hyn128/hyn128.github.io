---
title: Observability와 Linux Server Monitoring
description: Metric·Log·Trace의 차이를 이해하고 Linux CPU, Memory, Disk, Network와 Service 상태를 점검해 Alert와 장애 대응 흐름으로 연결합니다
date: 2026-10-09
series: CloudNative
tags:
  - CloudNative
  - AutoEverSW
  - Monitoring
---

## 요약

---

> Monitoring은 이미 정의한 상태와 장애 징후를 지속해서 확인하는 활동이고, Observability는 System이 내보내는 Metric, Log와 Trace를 연결해 예상하지 못한 문제의 원인을 추론할 수 있는 능력입니다. Linux Server에서는 CPU, Memory, Disk, Network뿐 아니라 Application 응답 시간과 Error Rate를 함께 확인해야 합니다. 수집한 신호는 Alert, Diagnosis, 조치와 Postmortem으로 이어져야 합니다.

## 1. Monitoring과 Observability

---

Observability는 System의 외부 출력만으로 내부 상태와 동작 원인을 얼마나 잘 이해할 수 있는지를 나타냅니다. Monitoring이 알려진 실패 조건을 감시하는 데 초점을 둔다면 Observability는 새로운 질문을 던지고 예측하지 못한 원인을 분석할 수 있도록 Telemetry를 준비하는 범위까지 포함합니다.

| 신호 | 의미 | 예시 |
| --- | --- | --- |
| Metrics | 시간에 따른 수치 측정값 | CPU 사용률, 요청 수, 응답 시간 |
| Logs | 특정 시점에 발생한 Event 기록 | Error Message, Access Log |
| Traces | 하나의 요청이 여러 Service를 거친 경로와 Span | API Gateway → Order → Payment |

Distributed Trace Context는 일반적으로 Service 사이의 HTTP Header나 Message Metadata로 전달합니다. Web Cookie 저장 자체가 Trace를 만드는 것은 아닙니다.

```text
Metrics: 언제, 얼마나 나빠졌는가
Logs: 그 시점에 어떤 Event가 있었는가
Traces: 요청이 어느 구간에서 지연되거나 실패했는가
```

## 2. Server Monitoring의 목적과 반복 구조

---

Server Monitoring은 Host와 Service의 상태를 지속해서 관찰하고 이상 징후를 탐지하는 활동입니다.

- 장애가 사용자에게 영향을 주기 전에 징후를 발견합니다.

- 장애 발생 시 영향 범위와 원인을 빠르게 좁힙니다.

- Resource 병목을 확인해 성능을 개선합니다.

- 추세를 이용해 증설 시점과 Capacity를 계획합니다.

운영 흐름은 한 번의 확인으로 끝나지 않습니다.

```text
Monitoring
  ↓
Alerting
  ↓
Diagnosis
  ↓
Action
  ↓
Review와 개선
  └────────→ Monitoring
```

CPU 사용률만 낮다고 Service가 정상인 것은 아닙니다. CPU가 20%여도 API 응답 시간이 5초라면 Database Lock, 외부 API, Disk I/O나 Thread 대기를 함께 확인해야 합니다.

## 3. Monitoring 대상

---

| 계층 | 주요 확인 항목 |
| --- | --- |
| Infrastructure | CPU, Memory, Disk, Network, Process, GPU |
| OS | Load Average, Disk I/O, File System, Kernel, System Log |
| Application | Response Time, Error Rate, Throughput |
| Database | Connection, Query Time, Lock, Cache, Replication |
| Container | Node, Pod, Restart, Resource, HPA 상태 |
| Cloud | Instance, Load Balancer, Auto Scaling, Managed Service |

Monitoring은 다음 네 관점을 함께 가져야 합니다.

| 관점 | 질문 |
| --- | --- |
| Availability | Service가 요청을 처리할 수 있는가 |
| Performance | 사용자가 허용 가능한 시간 안에 응답받는가 |
| Resources | CPU, Memory, Disk와 Network에 병목이 있는가 |
| Business·Service | 사용자의 핵심 업무 흐름이 실제로 성공하는가 |

## 4. CPU와 Load Average

---

| 지표 | 의미 |
| --- | --- |
| CPU Usage | CPU가 Busy 상태인 시간 비율 |
| User·System | Application Code와 Kernel Code에서 사용한 시간 |
| I/O Wait | CPU가 Block Device I/O 완료를 기다린 시간 |
| Load Average | 실행 중이거나 실행·I/O를 기다리는 Task 수의 평균 |

Load Average는 CPU 사용률과 같은 값이 아닙니다. CPU Core 수와 함께 보고, I/O Wait가 높으면 CPU 증설보다 Storage 지연을 먼저 확인합니다.

```bash
# [LINUX-HOST]
top
mpstat -P ALL 1 5
vmstat 1 5
```

Test용 CPU 부하는 Process ID를 기록해 반드시 종료합니다.

```bash
# [LINUX-HOST]
yes >/dev/null &
load_pid=$!

top

kill "$load_pid"
wait "$load_pid" 2>/dev/null || true
```

운영 Server에서는 Service 영향과 Autoscaling·Alert 동작을 확인한 뒤 격리된 Test 환경에서 실행합니다.

## 5. Memory

---

Linux는 남는 Memory를 Page Cache와 Buffer로 사용합니다. `free`가 작다는 이유만으로 Memory 부족이라고 판단하지 않고 새로운 Process가 사용할 수 있는 `available`을 확인합니다.

```bash
# [LINUX-HOST]
free -h
vmstat 1 5
```

| 항목 | 확인할 내용 |
| --- | --- |
| `available` | Swap 없이 새 Workload에 제공 가능한 추정 Memory |
| Swap 사용량 | 지속적인 증가와 Swap In·Out 발생 여부 |
| Process RSS | 실제 Memory를 많이 사용하는 Process |
| OOM Log | Kernel OOM Killer가 Process를 종료했는지 |

```bash
# [LINUX-HOST]
journalctl -k --grep='Out of memory\|Killed process'
```

Memory 부족은 응답 지연, Swap 증가와 OOM Kill로 이어질 수 있습니다. 순간값보다 시간에 따른 추세를 확인합니다.

## 6. Disk와 File System

---

Disk 문제는 공간 부족과 I/O 성능 저하를 구분합니다.

```bash
# [LINUX-HOST]
df -h
lsblk
iostat -xz 1 5

sudo du -sh /home/<MANAGED_USER> 2>/dev/null
sudo find /home/<MANAGED_USER> \
  -type f -size +5M -exec ls -lh {} \;
```

| 확인 영역 | 주요 지표 |
| --- | --- |
| File System 용량 | 사용률, 남은 Byte와 Inode |
| Disk I/O | Read·Write Throughput, IOPS, Latency와 Queue |
| 증가 원인 | Log, Database File, Upload와 Temporary File |

다음 값은 예제 운영 기준이며 모든 System에 동일하게 적용하지 않습니다.

| 사용률 | 예제 상태 |
| --- | --- |
| 70% 미만 | 정상 |
| 70~80% | 관찰 |
| 80~90% | Warning |
| 90% 초과 | Critical |
| 95% 초과 | 즉시 용량 확보와 증가 원인 조치 |

증가 속도, 예상 소진 시점, File System 특성과 자동 확장 가능 여부를 반영해 Threshold를 조정합니다.

## 7. Network와 Socket

---

| 지표 | 의미 |
| --- | --- |
| Traffic | 송수신 Byte와 Packet 양 |
| Error·Drop | Interface Error, Queue 또는 Network 혼잡 징후 |
| Connection | `ESTABLISHED`, `TIME_WAIT`와 Listen Socket 수 |
| Latency | 목적지까지의 왕복 지연과 변동 |

```bash
# [LINUX-HOST]
ip -s link
ss -s
ss -lntup
sar -n DEV 1 5
ping -c 3 <TARGET_IP>
```

`ping`은 ICMP Echo 요청과 응답을 확인하며 TCP Handshake를 검사하지 않습니다. Application Port는 `nc -vz <HOST> <PORT>` 또는 실제 Protocol Client로 확인합니다.

## 8. Log와 Process

---

```bash
# [LINUX-HOST]
systemctl --failed
journalctl -p warning..alert --since '30 minutes ago'
dmesg --level=err,warn
```

`journalctl`은 systemd Journal을, `dmesg`는 Kernel Ring Buffer를 확인합니다. Application Log와 함께 시간대를 맞추고 동일한 장애 시점의 Metric·Trace를 연결합니다.

## 9. Alert 설계

---

모든 Metric에 Alert를 만들면 중요한 신호가 반복 알림에 묻히는 Alert Fatigue가 발생합니다.

- 사용자 영향이나 즉시 조치 가능성이 있는 조건을 우선합니다.

- 단일 순간값보다 지속 시간과 여러 신호를 함께 평가합니다.

- Warning과 Critical의 의미, 담당자와 조치 절차를 정의합니다.

- 배포, 장애 조치와 Maintenance 중에는 Silence 또는 억제 정책을 사용합니다.

예를 들어 CPU 90%가 5분 이상 지속되면서 응답 시간과 Error Rate가 증가할 때 Warning을 발생시킬 수 있습니다. Disk는 사용률뿐 아니라 증가 속도와 남은 시간을 함께 보는 편이 효과적입니다.

## 10. 장애 대응 Process

---

| 단계 | 수행 내용 |
| --- | --- |
| Detect | Metric, Log, Trace와 사용자 신고로 이상 탐지 |
| Triage | 영향 Service, 사용자 범위와 심각도 판단 |
| Diagnose | 신호를 시간순으로 연결해 원인 후보 축소 |
| Mitigate | Traffic 우회, Rollback, Scale-out 등 긴급 조치 |
| Recover | Service Level과 사용자 흐름 정상화 확인 |
| Postmortem | 원인, 대응 과정과 재발 방지 Action 정리 |

긴급 조치와 근본 원인 해결을 구분합니다. Process 재시작으로 일시 복구됐더라도 Memory Leak이나 잘못된 배포 원인을 별도로 수정해야 합니다.

## 11. Dashboard 구성

---

```text
상단: Availability · Error Rate · Response Time · Throughput
중단: CPU · Memory · Disk · Network
하단: Process · Container · Kubernetes Object 상태
```

색을 많이 사용하는 것보다 정상 범위, 현재 이상과 시간 추세를 한눈에 구분할 수 있는 배치가 중요합니다. Dashboard는 실제 장애 대응 순서와 담당자의 질문을 기준으로 구성합니다.

## 12. Hybrid Monitoring Architecture

---

```text
On-Premise Linux → Node Exporter → Prometheus
Amazon EC2      → CloudWatch Agent → CloudWatch
Kubernetes      → Exporter·kube-state-metrics → Prometheus

Prometheus → Grafana
Prometheus → Alertmanager → Email·Chat·On-call
CloudWatch Alarm → SNS
```

CloudWatch의 EC2 기본 Metric, Agent와 Alarm은 [CloudWatch Metric·Logs·Alarm과 SNS 알림]({% post_url 2026-10-06-cloud-18-cloudwatch-metrics-logs-alarm-sns %})에서 다뤘습니다. Hybrid 환경에서는 시간 동기화, 공통 Label, Service 이름과 Alert 우선순위를 통일해야 서로 다른 Platform의 신호를 연결할 수 있습니다.

> **최종 정리**
> - Observability는 Metric, Log와 Trace를 연결해 예상하지 못한 문제를 분석할 수 있는 능력입니다.
>
> - Linux Memory는 `free`보다 `available`, Disk는 용량과 I/O 지연을 구분해 확인합니다.
>
> - `ping`은 ICMP 확인이며 Application Port와 TCP 상태는 별도 도구로 검증합니다.
>
> - Alert는 사용자 영향, 지속 시간과 조치 가능성을 기준으로 설계합니다.
>
> - Monitoring 결과는 Diagnosis, 조치, 복구 확인과 Postmortem으로 이어져야 합니다.

## 참고 자료

---

- [OpenTelemetry Observability 개요](https://opentelemetry.io/docs/concepts/observability-primer/)

- [OpenTelemetry Signal](https://opentelemetry.io/docs/concepts/signals/)

- [Prometheus 개요](https://prometheus.io/docs/introduction/overview/)
