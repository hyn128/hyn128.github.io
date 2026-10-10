---
title: Kubernetes Monitoring과 Prometheus Architecture
description: Kubernetes Resource Metrics Pipeline과 Prometheus Full Monitoring Pipeline을 구분하고 Node Exporter, kube-state-metrics, Alertmanager와 Operator의 역할을 정리합니다
date: 2026-10-09
series: CloudNative
tags:
  - CloudNative
  - AutoEverSW
  - Monitoring
---

## 요약

---

> Kubernetes Pod와 IP는 계속 생성·교체되므로 고정 Host 목록보다 API 기반 Service Discovery와 선언형 Monitoring 구성이 필요합니다. Metrics Server는 HPA와 `kubectl top`에 필요한 현재 CPU·Memory Resource Metric을 제공하고, Prometheus는 Exporter가 공개한 Metric을 주기적으로 수집해 시계열로 저장합니다. Grafana는 조회 결과를 시각화하고 Alertmanager는 Prometheus가 보낸 Alert를 Grouping·Deduplication·Routing합니다.

## 1. 고정 Host와 Kubernetes의 차이

---

| 구분 | 전통적인 Host 환경 | Kubernetes 환경 |
| --- | --- | --- |
| 대상 수명 | Host가 장기간 유지 | Pod가 생성·교체·삭제됨 |
| 주소 | 비교적 고정된 Host IP | Pod 재생성 시 IP 변경 가능 |
| 설정 | Host 목록과 정적 설정 사용 가능 | Label과 API 기반 Discovery 필요 |
| 상태 | Process 중심 | Node, Pod, Deployment와 Service 상태 연결 |

Kubernetes Monitoring은 Control Plane의 Desired State와 Worker Node의 실제 실행 상태를 함께 봐야 합니다.

```text
Control Plane
├── API Server의 Object 상태
├── Scheduler·Controller 동작
└── Desired Replica와 Condition

Worker Node
├── kubelet과 Container Runtime
├── Node CPU·Memory·Disk·Network
└── 실제 Pod Resource와 Restart
```

## 2. Resource Metrics Pipeline

---

Resource Metrics Pipeline은 HPA, VPA와 `kubectl top`에 필요한 최소 CPU·Memory Metric을 제공합니다.

```text
Container Runtime·cgroup
  ↓
kubelet의 Resource Metric Endpoint
  ↓ Pull
Metrics Server
  ↓ metrics.k8s.io API
API Server
  ├── kubectl top
  └── HPA·VPA
```

| 구성 요소 | 역할 |
| --- | --- |
| cAdvisor·CRI Metric | Container Resource 사용량 제공 |
| kubelet | Node와 Pod Resource 통계를 인증된 Endpoint로 공개 |
| Metrics Server | 각 Node의 kubelet에서 CPU·Memory Metric 수집·집계 |
| Metrics API | `metrics.k8s.io`로 현재 Resource Metric 제공 |

Metrics Server는 최신 값 중심의 짧은 수명 Pipeline입니다. 장기 추세 저장, 임의 Application Metric Query와 Alert를 위한 완전한 Monitoring System이 아닙니다.

```bash
# [CONTROL-PLANE-CLIENT]
kubectl top nodes
kubectl top pods --all-namespaces
```

명령이 실패하면 Metrics Server Pod, APIService `v1beta1.metrics.k8s.io`와 kubelet 통신을 확인합니다.

## 3. Full Monitoring Pipeline

---

Prometheus 기반 Pipeline은 여러 Target의 Metric을 수집해 시계열 Database에 보관합니다.

```text
Worker Node ─ Node Exporter ─┐
Kubernetes API ─ kube-state-metrics ─┤
Application ─ /metrics ─────┤
Batch Job ─ Pushgateway ────┘
                              ↓ Pull
                         Prometheus
                         ├── PromQL
                         ├── Recording·Alerting Rule
                         ├── Grafana
                         └── Alertmanager → Notification
```

Prometheus는 Target의 HTTP Metric Endpoint를 Pull 방식으로 Scrape합니다. Target은 정적 설정 또는 Kubernetes Service Discovery로 찾습니다.

## 4. 도구의 역할

---

| 도구 | 중심 역할 | 운영 형태 |
| --- | --- | --- |
| Prometheus | Metric 수집, 시계열 저장, PromQL과 Rule 평가 | 직접 운영 또는 Managed Service |
| Grafana | 여러 Data Source의 Dashboard와 시각화 | 직접 운영 또는 Managed Service |
| Datadog | Infrastructure·Log·APM 통합 Observability | SaaS |
| New Relic | Application Performance와 Telemetry 분석 | SaaS |
| InfluxDB | 범용 시계열 Database | 직접 운영, Cloud와 Enterprise |
| Kibana | Elasticsearch Data 검색과 시각화 | Elastic Stack |
| Chronograf | InfluxDB Data 탐색과 시각화 | InfluxDB 생태계 |

제품 기능, License와 요금은 Version과 계약에 따라 바뀝니다. 선택할 때는 다음을 비교합니다.

- Data를 조직 외부 Service에 저장할 수 있는가

- 수집 대상과 Data 양에 따른 비용을 감당할 수 있는가

- Metric, Log와 Trace를 하나의 Platform에서 연결해야 하는가

- Monitoring Platform 자체의 Backup과 고가용성을 운영할 수 있는가

## 5. Prometheus Server

---

Prometheus Server는 다음 기능을 담당합니다.

- Service Discovery와 설정으로 Scrape Target을 찾습니다.

- Target의 Metric Endpoint를 주기적으로 Pull합니다.

- Sample을 Label 기반 시계열로 저장합니다.

- PromQL Query, Recording Rule과 Alerting Rule을 평가합니다.

- Target 상태와 Query를 확인하는 Web UI와 API를 제공합니다.

Prometheus의 Local Storage는 각 Server가 독립적으로 동작하는 구조입니다. 장기 보관, 전역 Query와 대규모 고가용성이 필요하면 Remote Write나 별도 장기 Storage Architecture를 검토합니다.

## 6. Node Exporter

---

Node Exporter는 Linux Node의 `/proc`, `/sys`와 OS Interface에서 CPU, Memory, File System, Disk와 Network Metric을 읽어 HTTP `/metrics` Endpoint로 공개합니다.

Kubernetes에서는 일반적으로 DaemonSet으로 배포해 각 Worker Node의 Host Metric을 수집합니다. Control Plane Node에도 배포하려면 해당 Node의 Taint와 Monitoring 정책에 맞는 Toleration이 필요할 수 있습니다. Managed Kubernetes에서는 Provider가 관리하는 Control Plane Host에 Node Exporter를 직접 배포할 수 없습니다.

```text
Node Exporter: Host OS Resource 상태
kube-state-metrics: Kubernetes API Object 상태
```

두 Component는 같은 정보를 중복 수집하는 것이 아닙니다.

## 7. kube-state-metrics

---

`kube-state-metrics`는 Kubernetes API Server를 Watch하고 Deployment, ReplicaSet, Pod, Node 같은 Object의 현재 상태를 Metric으로 변환해 `/metrics`로 공개합니다.

| 예시 질문 | Metric 영역 |
| --- | --- |
| 원하는 Replica와 사용 가능한 Replica가 다른가 | Deployment State |
| Pod가 반복해서 Restart됐는가 | Pod Container Status |
| Node가 Ready Condition인가 | Node Status |
| Resource Request와 Limit가 어떻게 설정됐는가 | Pod Container Resource Spec |

Node의 실제 CPU 사용률을 읽는 Component가 아닙니다. API Object의 Spec과 Status를 Metric으로 나타냅니다.

## 8. Alertmanager

---

Alerting Rule은 Prometheus가 평가합니다. 조건이 충족되면 Prometheus가 Alert를 Alertmanager에 보내고 Alertmanager는 다음 작업을 수행합니다.

- 같은 원인의 Alert를 Grouping합니다.

- 중복 Notification을 제거합니다.

- Label에 따라 Email, Chat 또는 On-call Receiver로 Routing합니다.

- Maintenance 동안 Silence를 적용합니다.

- 상위 Alert가 발생했을 때 하위 Alert Notification을 Inhibition합니다.

Alertmanager가 Metric을 직접 수집하거나 Alert 조건을 평가하는 것은 아닙니다.

## 9. Pushgateway

---

Pushgateway는 Prometheus가 실행 중에 Scrape하기 어려운 짧은 Service-level Batch Job의 결과를 임시로 노출합니다.

```text
Short-lived Batch Job
  ↓ Push
Pushgateway
  ↓ Pull
Prometheus
```

일반 Host나 장기 Service의 Metric을 Pushgateway로 보내지 않습니다. Pushgateway는 Producer가 사라져도 Series를 자동 삭제하지 않으므로 오래된 Metric 수명 주기를 별도로 관리해야 합니다. Host 단위 Batch 결과는 Node Exporter Textfile Collector가 더 적합할 수 있습니다.

## 10. Prometheus Operator Pattern

---

Prometheus Operator는 Kubernetes Custom Resource를 Watch하고 Prometheus와 Alertmanager의 배포·설정을 Desired State에 맞게 조정합니다.

| Custom Resource | 역할 |
| --- | --- |
| `Prometheus` | Version, Replica, Storage와 Retention 등 Instance 상태 선언 |
| `Alertmanager` | Alertmanager Instance 상태 선언 |
| `ServiceMonitor` | Label로 Service Endpoint를 찾아 Scrape 방법 정의 |
| `PodMonitor` | Service 없이 Pod Label로 Target과 Scrape 방법 정의 |
| `PrometheusRule` | Recording Rule과 Alerting Rule 정의 |
| `AlertmanagerConfig` | Receiver, Route와 Inhibition 설정 분리 |

```text
사용자: ServiceMonitor·PrometheusRule 생성
  ↓
Kubernetes API Server
  ↓ Watch
Prometheus Operator
  ↓ Reconcile
Prometheus Scrape 설정·Rule과 Workload 갱신
  ↓
Prometheus가 새 Target Scrape·Rule 평가
```

Operator는 Label Selector를 이용해 `ServiceMonitor`, `PodMonitor`와 Rule을 Prometheus Instance에 연결합니다. 설정이 바뀌면 필요한 구성을 Reconcile하고 Prometheus가 재시작 없이 동적으로 반영할 수 있도록 관리합니다.

## 11. Control Plane과 Worker 관점

---

Monitoring Resource를 적용했을 때 역할은 다음처럼 이어집니다.

1. 관리자가 API Server에 `ServiceMonitor` 또는 `PrometheusRule`을 제출합니다.

2. API Server가 Desired State를 저장하고 Prometheus Operator가 변경을 감지합니다.

3. Operator가 Prometheus 설정과 관련 Workload를 Reconcile합니다.

4. Scheduler가 Monitoring Pod의 실행 Node를 선택합니다.

5. 선택된 Worker의 kubelet이 Container Runtime을 통해 Pod를 실행합니다.

6. Prometheus가 Worker의 Exporter와 Application Metric Endpoint를 Scrape합니다.

7. Prometheus Rule이 조건을 충족하면 Alertmanager로 Alert를 전달합니다.

Control Plane의 API Object 상태와 Worker의 Host·Container Resource Metric을 함께 봐야 `Pod가 원하는 수만큼 존재하지 않는 이유`와 `실행 중인 Pod가 느린 이유`를 구분할 수 있습니다.

## 12. Monitoring 설계 점검

---

- Metrics Server와 Prometheus의 목적을 구분했는가

- Node Exporter와 kube-state-metrics가 제공하는 상태를 구분했는가

- Label Cardinality가 불필요하게 증가하지 않는가

- Prometheus Retention과 Storage 용량을 계산했는가

- Alert에 담당자, 우선순위와 대응 절차가 연결됐는가

- Monitoring Component 자체의 Resource, 고가용성과 Backup을 설계했는가

- Metric Endpoint와 Dashboard 접근을 인증·Network Policy로 제한했는가

> **최종 정리**
> - Metrics Server는 HPA와 `kubectl top`에 필요한 현재 CPU·Memory Resource Metric을 제공합니다.
>
> - Prometheus는 Exporter와 Application Endpoint의 Metric을 Pull해 시계열로 저장합니다.
>
> - Node Exporter는 Host 상태를, kube-state-metrics는 Kubernetes API Object 상태를 공개합니다.
>
> - Prometheus가 Alert Rule을 평가하고 Alertmanager가 Notification을 Grouping·Deduplication·Routing합니다.
>
> - Prometheus Operator는 CRD와 Label Selector로 Monitoring Desired State를 Reconcile합니다.

## 참고 자료

---

- [Kubernetes Resource Metrics Pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)

- [Kubernetes Resource Monitoring 도구](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)

- [Prometheus Architecture](https://prometheus.io/docs/introduction/overview/)

- [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)

- [Prometheus Operator](https://prometheus-operator.dev/docs/getting-started/introduction/)

- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)

- [Pushgateway 사용 기준](https://prometheus.io/docs/practices/pushing/)
