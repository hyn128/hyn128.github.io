---
title: kube-prometheus-stack 설치와 Grafana·Prometheus 검증
description: Helm으로 kube-prometheus-stack을 설치하고 Kubernetes Monitoring Component, Grafana Dashboard, Prometheus Target과 Storage 상태를 검증합니다
date: 2026-10-12
series: CloudNative
tags:
  - CloudNative
  - AutoEverSW
  - Monitoring
---

## 요약

---

> `kube-prometheus-stack`은 Prometheus Operator를 중심으로 Prometheus, Alertmanager, Grafana, kube-state-metrics와 Node Exporter를 함께 설치하는 Helm Chart입니다. 설치 성공 여부는 Pod가 `Running`인지 확인하는 데서 끝나지 않습니다. Helm Release, Workload, Prometheus Target, Grafana Data Source와 Storage 상태를 순서대로 확인해야 합니다.

## 1. Helm Chart와 Release

---

> **Chart**는 Kubernetes Resource를 생성하기 위한 Template, 기본값과 Metadata를 묶은 Package입니다.

> **Release**는 Chart를 특정 Cluster와 Namespace에 설치한 Instance입니다. 같은 Chart도 Release 이름과 설정을 달리해 여러 번 설치할 수 있습니다.

이번 실습의 이름은 다음과 같습니다.

| 항목 | 값 |
| --- | --- |
| Chart | `prometheus-community/kube-prometheus-stack` |
| Release | `prometheus-stack` |
| Namespace | `monitoring` |

Helm은 Chart를 Rendering한 뒤 Kubernetes API Server에 Resource 생성을 요청합니다. Helm 자체가 Monitoring Data를 수집하는 것은 아닙니다.

## 2. 설치되는 주요 구성 요소

---

| 구성 요소 | 역할 | 일반적인 Workload |
| --- | --- | --- |
| Prometheus Operator | Monitoring CRD를 Watch하고 Prometheus 구성을 Reconcile합니다. | Deployment |
| Prometheus | Target의 Metric을 수집·저장하고 PromQL과 Rule을 평가합니다. | StatefulSet |
| Alertmanager | Alert를 Grouping·Deduplication하고 Receiver로 Routing합니다. | StatefulSet |
| Grafana | Prometheus Data Source를 Dashboard로 시각화합니다. | Deployment |
| kube-state-metrics | Kubernetes API Object의 Spec과 Status를 Metric으로 공개합니다. | Deployment |
| Node Exporter | 각 Node의 Linux Host Metric을 공개합니다. | DaemonSet |

Chart Version에 따라 생성되는 Resource 이름과 Container 구성이 달라질 수 있습니다. 명령에서 사용하는 Service 이름은 설치 후 `kubectl get service -n monitoring`으로 확인합니다.

## 3. Control Plane과 Worker의 동작

---

```text
관리 Host에서 Helm 실행
  ↓
API Server에 CRD·Deployment·StatefulSet·DaemonSet 등록
  ↓
Prometheus Operator가 Monitoring Resource Reconcile
  ↓
Scheduler가 실행할 Node 선택
  ↓
각 Worker의 kubelet이 Pod 실행
  ├── Node Exporter: Node마다 실행
  ├── Prometheus: Target Metric 수집
  ├── kube-state-metrics: API Object 상태 공개
  └── Grafana: Prometheus Data 시각화
```

Node Exporter는 DaemonSet이므로 Scheduling 조건을 만족하는 각 Node에서 실행됩니다. Prometheus와 Alertmanager는 StatefulSet으로 배포되며 Storage 설정을 추가하면 Pod가 교체돼도 PVC를 다시 연결할 수 있습니다.

## 4. 실행 위치와 사전 조건

---

| 표기 | 실행 위치 | 주요 작업 |
| --- | --- | --- |
| `[ADMIN]` | `kubectl`과 Helm을 사용할 관리 Host | 설치, 상태 확인과 Port Forwarding |
| `[BROWSER]` | 관리자가 사용하는 Browser | Grafana Dashboard와 Prometheus Target 확인 |

Cluster 연결과 Node 상태를 먼저 확인합니다.

```bash
# [ADMIN]
kubectl cluster-info
kubectl get nodes -o wide
kubectl auth can-i create namespaces
kubectl auth can-i create customresourcedefinitions.apiextensions.k8s.io
```

최신 Chart를 설치하려면 Cluster Version과 Chart의 지원 범위도 확인해야 합니다.

## 5. Helm 설치와 확인

---

공식 설치 Script를 바로 Pipe로 실행하지 않고 File로 내려받아 확인한 뒤 실행합니다.

```bash
# [ADMIN]
curl -fsSL \
  -o get_helm.sh \
  https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

less get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh

helm version
```

`curl | bash`는 내려받은 내용을 확인하기 전에 실행합니다. 학습 환경에서도 실행할 Script의 출처와 내용을 확인하는 절차를 유지합니다.

이미 Helm이 설치돼 있다면 Script를 다시 실행하지 않고 `helm version`만 확인합니다.

## 6. Chart Repository 준비

---

Prometheus Community Repository를 등록하고 Local Index를 갱신합니다.

```bash
# [ADMIN]
helm repo add \
  prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
helm search repo prometheus-community/kube-prometheus-stack
```

| 명령 | 변경·확인 대상 |
| --- | --- |
| `helm repo add` | Repository 이름과 URL을 Helm 설정에 등록합니다. |
| `helm repo update` | Chart Version과 Metadata Index를 갱신합니다. |
| `helm search repo` | 설치 가능한 Chart와 Version을 확인합니다. |

재현 가능한 설치가 필요하면 검증한 Chart Version을 기록하고 `helm install --version <CHART_VERSION>`으로 고정합니다.

## 7. kube-prometheus-stack 설치

---

Namespace를 생성하고 `prometheus-stack`이라는 Release 이름으로 Chart를 설치합니다.

```bash
# [ADMIN]
kubectl create namespace monitoring

helm install prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

Namespace가 이미 존재하면 `AlreadyExists`가 출력될 수 있습니다. 이 경우 Namespace 상태를 확인한 뒤 Helm 설치만 실행합니다.

설치 명령은 여러 CRD와 Workload를 생성합니다. 일부 Pod는 Image Pull과 초기화에 시간이 필요하므로 즉시 모든 Pod가 `Running`이 아닐 수 있습니다.

## 8. Helm Release와 Workload 검증

---

Helm과 Kubernetes 두 관점에서 상태를 확인합니다.

```bash
# [ADMIN]
helm status prometheus-stack --namespace monitoring
helm list --namespace monitoring

kubectl get pods -n monitoring -o wide
kubectl get deployments,statefulsets,daemonsets -n monitoring
kubectl get services -n monitoring
```

확인 기준은 다음과 같습니다.

- Helm Release의 상태가 `deployed`인가

- Prometheus Operator와 Grafana Deployment가 사용 가능한 Replica를 확보했는가

- Prometheus와 Alertmanager StatefulSet Pod가 `Ready`인가

- Node Exporter Pod가 대상 Node마다 실행되는가

- kube-state-metrics Pod가 실행되고 Service Endpoint가 생성됐는가

Pod가 `Pending`, `CrashLoopBackOff` 또는 `ImagePullBackOff` 상태라면 Event와 Log를 확인합니다.

```bash
# [ADMIN]
kubectl get events -n monitoring \
  --sort-by=.metadata.creationTimestamp

kubectl describe pod <POD_NAME> -n monitoring
kubectl logs <POD_NAME> -n monitoring --all-containers
```

## 9. Grafana 접근과 Dashboard 확인

---

### 9.1 Port Forwarding

Grafana Service를 Local Port `3000`으로 전달합니다.

```bash
# [ADMIN]
kubectl port-forward \
  service/prometheus-stack-grafana \
  3000:80 \
  --namespace monitoring
```

기본 Bind Address는 `127.0.0.1`이므로 관리 Host 외부에 Port가 공개되지 않습니다. `--address 0.0.0.0`을 추가하면 다른 Host에서도 접근할 수 있으므로 방화벽과 접근 대상을 제한하지 않은 상태에서는 사용하지 않습니다.

### 9.2 초기 Login 정보 확인

ID는 `admin`입니다. Password는 Secret에서 조회하되 문서, Screenshot, Shell Script와 Git Commit에 기록하지 않습니다.

```bash
# [ADMIN]
kubectl get secret prometheus-stack-grafana \
  --namespace monitoring \
  --output jsonpath='{.data.admin-password}' \
  | base64 --decode
echo
```

조회한 초기 Password는 공유 Credential로 계속 사용하지 않고 운영 구성에서는 Secret 관리 체계와 Grafana 인증 설정을 적용합니다.

### 9.3 Kubernetes Dashboard 확인

Browser에서 `http://127.0.0.1:3000`에 접속합니다.

1. `Dashboards`를 선택합니다.

2. `Browse`를 선택합니다.

3. `Kubernetes / Compute Resources / Cluster` Dashboard를 엽니다.

4. Node, Namespace, Pod Resource Graph에 Data가 표시되는지 확인합니다.

## 10. Prometheus Web UI와 Target 확인

---

Prometheus Service를 Local Port `9090`으로 전달합니다.

```bash
# [ADMIN]
kubectl port-forward \
  service/prometheus-stack-kube-prom-prometheus \
  9090:9090 \
  --namespace monitoring
```

Browser에서 `http://127.0.0.1:9090`에 접속합니다. Prometheus UI의 `Status → Targets`에서 다음을 확인합니다.

- Target이 검색되는가

- `Health`가 `UP`인가

- `Last Scrape`가 계속 갱신되는가

- Error 열에 Scrape 실패 원인이 표시되는가

이번 범위는 설치와 연결 상태 확인까지입니다. PromQL 문법, Custom Application Scrape, `ServiceMonitor`와 `PodMonitor` 작성은 별도 학습 범위입니다.

## 11. NodePort 실습 선택지

---

Port Forwarding을 유지할 수 없는 폐쇄된 실습 Network에서는 Grafana와 Prometheus Service를 NodePort로 변경할 수 있습니다.

```bash
# [ADMIN]
kubectl patch service prometheus-stack-grafana \
  --namespace monitoring \
  --type merge \
  --patch '{"spec":{"type":"NodePort"}}'

kubectl patch service prometheus-stack-kube-prom-prometheus \
  --namespace monitoring \
  --type merge \
  --patch '{"spec":{"type":"NodePort"}}'

kubectl get services -n monitoring
```

NodePort는 Node Network에 접근할 수 있는 Host에 Web UI를 노출합니다. 인증, 방화벽과 Security Group을 확인하지 않은 채 Internet에 공개하지 않습니다.

실습이 끝나면 Service를 ClusterIP로 복구합니다.

```bash
# [ADMIN]
kubectl patch service prometheus-stack-grafana \
  --namespace monitoring \
  --type merge \
  --patch '{"spec":{"type":"ClusterIP"}}'

kubectl patch service prometheus-stack-kube-prom-prometheus \
  --namespace monitoring \
  --type merge \
  --patch '{"spec":{"type":"ClusterIP"}}'
```

운영 환경의 Web UI는 TLS와 인증을 적용한 Ingress, Gateway 또는 조직의 Private Access 경로를 사용합니다.

## 12. Storage 확인

---

Storage 설정을 별도로 지정하지 않은 실습 구성은 Pod 재생성 시 Data가 유지되지 않을 수 있습니다. 먼저 StorageClass와 PVC 상태를 확인합니다.

```bash
# [ADMIN]
kubectl get storageclass
kubectl get persistentvolumeclaims -n monitoring
kubectl get statefulsets -n monitoring
```

| 확인 결과 | 의미 |
| --- | --- |
| PVC가 없음 | Chart 설정에서 영구 Storage를 사용하지 않는 구성일 수 있습니다. |
| PVC가 `Pending` | 요청 조건을 만족하는 StorageClass나 Volume을 찾지 못했습니다. |
| PVC가 `Bound` | PVC가 PV에 연결됐습니다. 실제 Mount와 Data 유지 여부를 추가로 확인합니다. |

운영 환경에서는 Chart의 `values.yaml`에 Prometheus와 Grafana Persistence, StorageClass, 요청 용량과 Retention을 명시합니다. NFS나 `local-path-provisioner`를 사용한다는 이유만으로 Backup과 고가용성이 자동으로 확보되지는 않습니다.

## 13. Resource 계획과 장애 점검

---

소규모 실습에서는 Node당 4GB 이상의 RAM을 계획값으로 둘 수 있지만 모든 Cluster에 적용되는 절대 최소 요구 사항은 아닙니다. 실제 요구량은 Node 수, Scrape Target 수, Scrape 주기, Retention, Dashboard와 Rule 수에 따라 달라집니다.

설치 후 다음 명령으로 실제 사용량과 Scheduling 실패를 확인합니다.

```bash
# [ADMIN]
kubectl top nodes
kubectl top pods -n monitoring
kubectl get events -n monitoring \
  --field-selector reason=FailedScheduling
```

`kubectl top`을 사용할 수 없다면 Resource Metrics Pipeline 상태를 별도로 확인해야 합니다.

Dashboard에 Data가 표시되지 않으면 다음 순서로 점검합니다.

1. Node Exporter Pod가 대상 Node마다 `Ready`인지 확인합니다.

2. kube-state-metrics Pod와 Service Endpoint를 확인합니다.

3. Prometheus `Status → Targets`에서 Scrape Error를 확인합니다.

4. Grafana의 Prometheus Data Source 연결 상태를 확인합니다.

5. 선택한 Dashboard의 시간 범위와 변수를 확인합니다.

6. Storage를 사용하는 경우 PVC가 `Bound`이고 Volume Mount Error가 없는지 확인합니다.

> **최종 정리**
> - Helm은 Chart를 Release로 설치하고 Kubernetes API Server에 Monitoring Resource를 등록합니다.
>
> - Node Exporter는 Node마다 실행되고 Prometheus Operator는 Monitoring Resource를 원하는 상태로 조정합니다.
>
> - 설치 검증은 Helm Release, Workload, Prometheus Target, Grafana Dashboard와 Storage 순서로 진행합니다.
>
> - Port Forwarding을 기본 접근 방식으로 사용하고 NodePort와 Ingress는 접근 제어 범위를 확인한 뒤 선택합니다.
>
> - RAM과 Storage 요구량은 고정된 숫자가 아니라 Target 수, Scrape 주기와 Retention을 기준으로 결정합니다.

## 참고 자료

---

- [Helm 설치](https://helm.sh/docs/intro/install/)

- [Helm Quickstart](https://helm.sh/docs/intro/quickstart/)

- [kube-prometheus-stack Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

- [Prometheus Operator](https://prometheus-operator.dev/docs/getting-started/introduction/)

- [Prometheus Target 상태](https://prometheus.io/docs/prometheus/latest/getting_started/)
