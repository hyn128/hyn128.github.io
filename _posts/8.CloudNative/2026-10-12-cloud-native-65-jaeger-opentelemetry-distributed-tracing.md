---
title: Jaeger와 OpenTelemetry로 Kubernetes 분산 추적
description: Jaeger v2 All-in-one과 OpenTelemetry로 Flask Application을 계측하고 Kubernetes에서 Trace를 전송·조회하는 과정을 정리합니다
date: 2026-10-12
series: CloudNative
tags:
  - CloudNative
  - AutoEverSW
  - Monitoring
---

## 요약

---

> 분산 추적은 하나의 요청이 여러 Service를 통과하는 과정을 Trace와 Span으로 기록합니다. Application은 OpenTelemetry SDK로 Trace를 생성하고 OTLP로 Jaeger Collector에 전송합니다. Jaeger는 Trace를 저장하며 Query Service와 UI는 수집된 요청의 호출 순서, 소요 시간과 Error 구간을 조회합니다.

## 1. 분산 추적이 필요한 이유

---

Microservice 환경에서는 하나의 사용자 요청이 여러 Service를 거칠 수 있습니다.

```text
Client
  ↓
Frontend
  ↓
Order API
  ├── Payment Service
  └── Inventory Service
```

전체 응답이 느리다는 사실만으로는 어느 Service에서 지연이 발생했는지 판단하기 어렵습니다. 분산 추적은 요청에 같은 Context를 전달하고 각 처리 구간을 Span으로 기록해 다음 질문에 답합니다.

- 요청이 어떤 Service를 거쳤는가

- 각 구간에서 얼마의 시간이 걸렸는가

- Error 또는 지연이 처음 발생한 구간은 어디인가

- Service 사이의 호출 관계는 어떻게 구성되는가

## 2. Trace와 Span

---

> **Trace**는 하나의 요청이 시작부터 종료될 때까지 거친 전체 처리 흐름입니다.

> **Span**은 Trace를 구성하는 개별 작업 단위입니다. Service 호출, Database Query와 특정 Business Logic을 각각 Span으로 기록할 수 있습니다.

> **Context**는 Trace ID와 현재 Span 정보를 담고 다음 Service로 전달되는 정보입니다. Context가 전달돼야 여러 Service가 생성한 Span을 하나의 Trace로 연결할 수 있습니다.

```text
Trace: 사용자 주문 요청
├── Span: Frontend 요청 처리
├── Span: Order API 처리
│   ├── Span: Payment Service 호출
│   └── Span: Inventory Service 호출
└── Span: 응답 반환
```

Trace ID는 전체 요청을 식별하고 Span ID는 개별 구간을 식별합니다. Parent Span ID는 호출 관계를 연결합니다.

## 3. Jaeger 구성 요소

---

| 구성 요소 | 역할 |
| --- | --- |
| OpenTelemetry SDK | Application에서 Trace와 Span을 생성하고 Context를 전달합니다. |
| Collector | OTLP로 전달된 Trace를 수신하고 처리한 뒤 Storage에 기록합니다. |
| Storage | Trace Data를 저장합니다. |
| Query Service | 조건에 맞는 Trace를 Storage에서 조회합니다. |
| Jaeger UI | Service, Operation과 시간 범위로 Trace를 검색하고 Timeline을 시각화합니다. |

이번 실습은 Collector, Query Service와 Memory Storage를 하나의 Process로 실행하는 Jaeger All-in-one 구성을 사용합니다.

```text
Flask Application
  ↓ OTLP gRPC :4317
Jaeger Collector
  ↓
Memory Storage
  ↓
Jaeger Query Service
  ↓ HTTP :16686
Jaeger UI
```

Memory Storage의 Trace는 Jaeger Pod가 재시작되면 사라집니다. All-in-one과 Memory Storage는 기능 확인을 위한 실습 구성이고 장기 보관이나 고가용성을 제공하는 운영 구성은 아닙니다.

## 4. Jaeger Version과 Port

---

Jaeger v1은 2025년 12월 31일에 지원이 종료됐습니다. `jaegertracing/all-in-one:1.57.0`과 `COLLECTOR_OTLP_ENABLED` 같은 v1 예제 대신 Jaeger v2 Image를 사용합니다.

| Port | Protocol | 용도 |
| --- | --- | --- |
| `4317` | gRPC | OpenTelemetry OTLP Trace 수신 |
| `4318` | HTTP | OpenTelemetry OTLP HTTP `/v1/traces` 수신 |
| `16686` | HTTP | Jaeger Query Service와 Web UI |

실습 Image는 결과가 임의로 바뀌지 않도록 `cr.jaegertracing.io/jaegertracing/jaeger:2.22.0`으로 고정합니다.

## 5. 실행 위치와 사전 조건

---

| 표기 | 실행 위치 | 주요 작업 |
| --- | --- | --- |
| `[ADMIN]` | `kubectl`이 설정된 관리 Host | Manifest 작성·적용, 상태 확인과 Port Forwarding |
| `[BUILD]` | Docker를 실행할 수 있는 Build Host | Flask Image Build와 Registry Push |
| `[BROWSER]` | 관리자가 사용하는 Browser | Jaeger UI에서 Trace 검색 |

다음 조건을 먼저 확인합니다.

```bash
# [ADMIN]
kubectl cluster-info
kubectl get nodes
kubectl auth can-i create deployments.apps --namespace observability
kubectl auth can-i create services --namespace observability
```

`yes`가 출력되지 않으면 해당 Namespace에 Deployment와 Service를 생성할 권한이 필요합니다.

## 6. Jaeger v2 All-in-one 배포

---

`jaeger-all-in-one.yaml`에 Namespace, Deployment와 두 Service를 작성합니다. 하나의 Jaeger Pod를 사용하지만 Trace 수집 경로와 UI 조회 경로를 구분하기 위해 Service를 분리합니다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: observability
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: observability
  labels:
    app.kubernetes.io/name: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: jaeger
  template:
    metadata:
      labels:
        app.kubernetes.io/name: jaeger
    spec:
      containers:
        - name: jaeger
          image: cr.jaegertracing.io/jaegertracing/jaeger:2.22.0
          ports:
            - name: otlp-grpc
              containerPort: 4317
            - name: otlp-http
              containerPort: 4318
            - name: query-http
              containerPort: 16686
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: observability
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: jaeger
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: otlp-grpc
    - name: otlp-http
      port: 4318
      targetPort: otlp-http
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: observability
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: jaeger
  ports:
    - name: query-http
      port: 16686
      targetPort: query-http
```

Manifest를 적용하고 Rollout 상태를 확인합니다.

```bash
# [ADMIN]
kubectl apply -f jaeger-all-in-one.yaml
kubectl rollout status deployment/jaeger -n observability
kubectl get pods,services,endpoints -n observability
```

`jaeger-collector` Endpoint에는 `4317`, `4318`이 표시되고 `jaeger-query` Endpoint에는 `16686`이 표시돼야 합니다.

## 7. Flask Application 계측

---

### 7.1 Package 구성

`requirements.txt`를 작성합니다.

```text
Flask
opentelemetry-api
opentelemetry-sdk
opentelemetry-exporter-otlp-proto-grpc
opentelemetry-instrumentation-flask
```

| Package | 역할 |
| --- | --- |
| `opentelemetry-api` | Trace와 Span을 생성하는 공통 API를 제공합니다. |
| `opentelemetry-sdk` | Tracer Provider, Processor와 Export 동작을 구현합니다. |
| `opentelemetry-exporter-otlp-proto-grpc` | Trace를 OTLP gRPC로 Collector에 전송합니다. |
| `opentelemetry-instrumentation-flask` | Flask 요청을 자동으로 계측합니다. |

### 7.2 Application 작성

`app.py`를 작성합니다.

```python
import time

from flask import Flask
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

resource = Resource.create({"service.name": "sample-python-api"})
provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(
    OTLPSpanExporter(
        endpoint="http://jaeger-collector.observability.svc.cluster.local:4317",
        insecure=True,
    )
)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)


@app.route("/")
def hello():
    with tracer.start_as_current_span("business-logic-span"):
        time.sleep(0.1)
        return "Hello Jaeger in Kubernetes!\n"


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

`service.name`은 Jaeger UI의 Service 목록에 표시할 논리적인 Application 이름입니다. Flask 자동 계측은 HTTP 요청 Span을 만들고 `business-logic-span`은 Application 내부 처리 구간을 별도 Span으로 기록합니다.

OTLP gRPC는 실습 Cluster 내부의 평문 통신이므로 `insecure=True`를 사용합니다. Cluster 밖이나 신뢰할 수 없는 Network를 통과한다면 TLS와 인증을 구성해야 합니다.

## 8. Container Image Build와 Push

---

`Dockerfile`을 작성합니다.

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8080

CMD ["python", "app.py"]
```

Registry 사용자 이름과 Tag를 실제 값으로 바꿔 Image를 Build하고 Push합니다. Password나 Access Token은 명령 인자, Source Code와 문서에 직접 작성하지 않습니다.

```bash
# [BUILD]
TRACE_IMAGE_REGISTRY_USER="<CONTAINER_REGISTRY_USERNAME>"
TRACE_IMAGE_TAG="<TAG>"

docker build \
  -t "${TRACE_IMAGE_REGISTRY_USER}/jaeger-sample-api:${TRACE_IMAGE_TAG}" .

docker login

docker push \
  "${TRACE_IMAGE_REGISTRY_USER}/jaeger-sample-api:${TRACE_IMAGE_TAG}"
```

`<TAG>`에는 `latest`보다 `1.0.0`이나 Commit SHA처럼 배포한 Image를 구분할 수 있는 값을 사용합니다.

## 9. Sample API 배포

---

`sample-api.yaml`을 작성합니다. `image` 값을 앞에서 Push한 Image로 변경해야 합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-api
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: sample-api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: sample-api
    spec:
      containers:
        - name: api
          image: <CONTAINER_REGISTRY_USERNAME>/jaeger-sample-api:<TAG>
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: sample-api
  namespace: observability
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: sample-api
  ports:
    - name: http
      port: 80
      targetPort: http
```

```bash
# [ADMIN]
kubectl apply -f sample-api.yaml
kubectl rollout status deployment/sample-api -n observability
kubectl get pods,services,endpoints -n observability -o wide
```

`ImagePullBackOff`가 발생하면 Image 이름과 Tag, Registry 공개 범위, `imagePullSecrets`를 확인합니다.

## 10. 요청을 생성하고 Trace 확인

---

### 10.1 Sample API 요청

첫 번째 Terminal에서 Sample API Service를 Local Port로 전달합니다.

```bash
# [ADMIN: TERMINAL 1]
kubectl port-forward service/sample-api 8080:80 -n observability
```

두 번째 Terminal에서 여러 번 요청을 보냅니다.

```bash
# [ADMIN: TERMINAL 2]
for request in 1 2 3 4 5; do
  curl --fail --silent --show-error http://127.0.0.1:8080/
done
```

Application이 Collector로 Trace를 전송하지 못하면 Pod Log를 확인합니다.

```bash
# [ADMIN]
kubectl logs deployment/sample-api -n observability
kubectl logs deployment/jaeger -n observability
```

### 10.2 Jaeger UI 접근

Jaeger UI는 ClusterIP로 유지하고 Port Forwarding으로 접근합니다.

```bash
# [ADMIN]
kubectl port-forward service/jaeger-query 16686:16686 -n observability
```

Browser에서 `http://127.0.0.1:16686`에 접속한 뒤 다음 순서로 확인합니다.

1. `Service`에서 `sample-python-api`를 선택합니다.

2. `Find Traces`를 실행합니다.

3. Trace를 선택해 Flask 요청 Span과 `business-logic-span`의 Parent·Child 관계를 확인합니다.

4. 각 Span의 시작 시각, Duration과 Tag를 확인합니다.

요청 직후 Service가 보이지 않으면 잠시 기다린 뒤 다시 조회하고 Application과 Jaeger Log에서 OTLP 전송 Error를 확인합니다.

## 11. NodePort 선택지와 주의 사항

---

Port Forwarding을 유지할 수 없는 실습 환경에서는 Sample API를 NodePort로 변경할 수 있습니다.

```bash
# [ADMIN]
kubectl patch service sample-api -n observability \
  --type merge \
  --patch '{"spec":{"type":"NodePort"}}'

kubectl get service sample-api -n observability
```

NodePort를 사용하면 Node Network에 접근 가능한 다른 Host도 Service에 접속할 수 있습니다. 방화벽과 접근 가능한 Network 범위를 확인해야 합니다. 실습이 끝나면 ClusterIP로 복구합니다.

```bash
# [ADMIN]
kubectl patch service sample-api -n observability \
  --type merge \
  --patch '{"spec":{"type":"ClusterIP"}}'
```

Jaeger UI는 Trace에 포함된 Service 이름, Route와 Attribute를 노출할 수 있으므로 인증 없이 외부에 공개하지 않습니다.

## 12. 운영 환경으로 확장할 때 확인할 항목

---

- OpenTelemetry Operator 또는 Jaeger Helm Chart로 수명 주기를 관리합니다.

- Memory Storage 대신 운영 요구 사항에 맞는 영구 Storage를 구성합니다.

- Collector와 Query Service를 분리하고 Replica, Resource와 고가용성을 설계합니다.

- OTLP Endpoint와 Jaeger UI에 TLS, 인증, Network Policy와 접근 제어를 적용합니다.

- Sampling 정책으로 수집량, Storage 사용량과 비용을 제어합니다.

- Trace Attribute에 Password, Token, 개인정보와 민감한 Request Body를 기록하지 않습니다.

> **최종 정리**
> - Trace는 하나의 요청 전체를 나타내고 Span은 요청을 구성하는 개별 처리 구간입니다.
>
> - OpenTelemetry SDK는 Flask 요청과 사용자 정의 작업을 계측해 OTLP로 Jaeger Collector에 전송합니다.
>
> - Jaeger All-in-one과 Memory Storage는 실습에 적합하지만 Pod 재시작 시 Trace가 사라집니다.
>
> - Application 요청을 생성한 뒤 Jaeger UI에서 Service, Trace와 Span의 Duration을 확인해야 전체 흐름을 검증할 수 있습니다.

## 참고 자료

---

- [Jaeger v2 Getting Started](https://www.jaegertracing.io/docs/2.22/getting-started/)

- [Jaeger Kubernetes 배포](https://www.jaegertracing.io/docs/2.22/deployment/kubernetes/)

- [Jaeger Download와 Version 상태](https://www.jaegertracing.io/download/)

- [OpenTelemetry Python Instrumentation](https://opentelemetry.io/docs/languages/python/instrumentation/)

- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
