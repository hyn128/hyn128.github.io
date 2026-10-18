---
title: Spring Boot Micrometer 계측과 Prometheus ServiceMonitor 연동
description: Actuator와 Micrometer로 Spring Boot Metric을 노출하고 Kubernetes ServiceMonitor를 통해 수집한 뒤 Grafana에서 확인합니다
date: 2026-10-13
series: CloudNative
tags:
  - CloudNative
  - AutoEverSW
  - Monitoring
---

## 요약

---

> Spring Boot Application은 Micrometer로 Metric을 기록하고 Actuator의 `/actuator/prometheus`에서 수집 가능한 형태로 공개합니다. Kubernetes에서는 ServiceMonitor가 Service Label과 Port 이름으로 수집 대상을 지정합니다. Prometheus가 각 Application Pod를 주기적으로 Scrape하면 Grafana에서 요청 횟수와 처리 시간을 확인할 수 있습니다.

## 1. Actuator와 Micrometer

---

| 구성 요소 | 역할 |
| --- | --- |
| Spring Boot Actuator | Health와 Metric 등 Application 내부 상태를 Endpoint로 공개합니다. |
| Micrometer | Monitoring Backend에 독립적인 Metric 계측 API를 제공합니다. |
| MeterRegistry | Counter, Gauge, Timer 등 Meter를 등록하고 관리합니다. |
| Prometheus Registry | Micrometer Metric을 Prometheus가 수집할 수 있는 형식으로 변환합니다. |

Actuator Endpoint를 노출하는 설정과 Prometheus Registry 의존성이 모두 필요합니다. 응답은 Content Negotiation에 따라 Prometheus Text 또는 OpenMetrics 형식이 될 수 있으므로 항상 OpenMetrics만 반환한다고 가정하지 않습니다.

## 2. Meter 종류와 Tag 설계

---

| Meter·통계 | 용도 | 예시 |
| --- | --- | --- |
| Counter | 발생 횟수를 누적 | 주문 완료, Error 발생, 방문 횟수 |
| Gauge | 현재 값을 관찰 | 활성 Connection, Cache 크기, Queue 길이 |
| Timer | 처리 횟수와 소요 시간을 기록 | HTTP 응답 시간, Database Query 시간 |
| DistributionSummary | 시간 이외의 값의 분포 기록 | 결제 금액, Payload 크기 |
| Histogram | 관측값을 Bucket 경계별로 집계 | 100ms·200ms 이내 요청 수 |
| Percentile | 관측값 분포의 특정 백분위 | p95·p99 응답 시간 |

Counter는 실행 중 증가하지만 Process가 재시작되면 초기화될 수 있습니다. Timer도 처리 횟수를 기록하므로 동일 작업의 횟수만 필요하다면 별도 Counter가 항상 필요한 것은 아닙니다.

다음은 주입받은 `registry`와 Application이 유지하는 `cacheMap`을 사용하는 개별 계측 예시입니다. 각 Class는 `io.micrometer.core.instrument`에서 가져옵니다.

```java
Counter.builder("order.success.total")
    .tag("shop", "branch-a")
    .register(registry)
    .increment();

Gauge.builder("cache.size", cacheMap, Map::size)
    .register(registry);

Timer.Sample sample = Timer.start(registry);
// 측정할 작업을 수행합니다.
sample.stop(registry.timer("api.latency"));

DistributionSummary.builder("order.amount")
    .register(registry)
    .record(55000);
```

Gauge는 관찰 대상 객체의 현재 값을 읽습니다. `cacheMap`은 지역 변수로 생성하고 버리는 객체가 아니라 Application이 참조를 유지하는 실제 Cache여야 합니다.

p95가 200ms라면 관측 요청의 약 95%가 200ms 이하라는 의미입니다. Timer 생성만으로 p95·p99나 Histogram Bucket이 자동 노출되지는 않습니다. Histogram 또는 Client-side Percentile 발행 설정이 필요하며 여러 Pod의 Percentile 값을 단순 평균해 전체 Percentile로 사용할 수는 없습니다.

### Cardinality 관리

> **Cardinality**는 Metric 이름과 Label 값 조합으로 생성되는 시계열의 수입니다.

사용자 ID, 주문 번호처럼 계속 늘어나는 값을 Tag로 지정하면 시계열 수가 급증합니다. `status=success|fail`, `method=card|transfer`처럼 값의 종류가 제한된 Tag를 사용합니다. 비밀번호, Token과 개인정보를 Tag에 넣지 않습니다.

## 3. 실행 환경과 Project 준비

---

| 표기 | 실행 위치 | 작업 |
| --- | --- | --- |
| [DEV] | JDK·Gradle Wrapper가 있는 개발 Host | Spring Boot 작성과 Local 검증 |
| [BUILD] | Docker를 사용할 Build Host | Image Build·Push |
| [ADMIN] | Cluster 접근 권한이 있는 kubectl 관리 Host | 배포, ServiceMonitor 적용, 상태 확인 |
| [BROWSER] | Port Forwarding을 실행한 Host의 Browser | Prometheus와 Grafana 확인 |

예제는 Java 17과 호환되는 Spring Boot 3.x Project, Gradle Groovy DSL을 기준으로 합니다. Project의 Spring Boot Plugin과 Gradle Wrapper는 서로 호환되는 Version을 사용합니다. 기존 Project가 다른 Java Version을 사용하면 Docker Build·Runtime Image도 함께 맞춥니다.

Spring Initializr에서 Spring Web을 포함해 Project를 생성합니다. Lombok은 선택할 수 있지만 아래 코드는 생성자를 직접 작성하므로 필수 의존성이 아닙니다.

`build.gradle`의 기존 `dependencies`에 다음 항목을 반영합니다. 이미 존재하는 Spring Web은 중복 추가하지 않습니다. Micrometer Version은 Spring Boot 의존성 관리에 맡깁니다.

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}

tasks.named('bootJar') {
    archiveFileName = 'app.jar'
}
```

`bootJar`의 출력 이름을 고정해 Dockerfile이 실행 가능한 JAR 하나만 복사하도록 합니다.

## 4. Actuator Endpoint 설정

---

`src/main/resources/application.yml`에 작성합니다.

```yaml
server:
  port: 8080

spring:
  application:
    name: prometheus-spring

management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
  metrics:
    tags:
      application: spring-boot-metrics-app
```

| 설정 | 의미 |
| --- | --- |
| `health` | Application Health Endpoint 노출 |
| `info` | 설정된 Application 정보 노출 |
| `prometheus` | `/actuator/prometheus` 노출 |
| `management.metrics.tags.application` | Metric에 공통 Application Label 추가 |

Endpoint 노출은 인증 설정과 별개입니다. Spring Security가 적용된 Project에서는 Prometheus 요청에 대한 접근 정책도 확인해야 합니다. Metric Endpoint에는 JVM·Application 내부 정보가 포함되므로 실습에서는 ClusterIP와 Local Port Forwarding으로 접근합니다.

## 5. Custom Metric Controller 작성

---

Application의 Component Scan 범위 안에 `SampleController.java`를 작성합니다. 예제 Package는 `com.example.metrics`이며 Main Application Class도 이 Package 또는 상위 Package에 배치합니다.

```java
package com.example.metrics;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import java.util.concurrent.ThreadLocalRandom;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.server.ResponseStatusException;

@RestController
public class SampleController {
    private final Counter customVisitCounter;
    private final Timer customProcessTimer;

    public SampleController(MeterRegistry registry) {
        this.customVisitCounter = Counter.builder("app.visitations.total")
            .description("Total visits to the sample endpoints")
            .tag("region", "region-a")
            .register(registry);
        this.customProcessTimer = Timer.builder("app.process.latency")
            .description("Time taken to process sample business logic")
            .register(registry);
    }

    @GetMapping("/hello")
    public String hello() {
        customVisitCounter.increment();
        return "Hello Micrometer & Kubernetes";
    }

    @GetMapping("/process")
    public String processWork(
            @RequestParam(name = "millis", defaultValue = "100") int millis) {
        if (millis < 0 || millis > 5000) {
            throw new ResponseStatusException(
                HttpStatus.BAD_REQUEST, "millis must be between 0 and 5000");
        }
        customVisitCounter.increment();
        return customProcessTimer.record(() -> {
            try {
                int sleepTime = millis + ThreadLocalRandom.current().nextInt(50);
                Thread.sleep(sleepTime);
            } catch (InterruptedException exception) {
                Thread.currentThread().interrupt();
                throw new ResponseStatusException(
                    HttpStatus.SERVICE_UNAVAILABLE, "Processing interrupted", exception);
            }
            return "Processed work in simulated time";
        });
    }
}
```

Spring이 자동 구성한 MeterRegistry를 생성자에서 주입받아 Meter를 한 번 등록합니다. 요청마다 새 Registry를 생성하지 않습니다.

`/hello`와 유효한 `/process` 요청 모두 방문 Counter를 증가시킵니다. `/process`만 Timer로 처리 시간을 측정하며, 이 Counter는 성공 건수만을 의미하지 않습니다. `millis` 범위 검사는 음수 Sleep과 과도한 대기를 막습니다. Sleep은 지연을 재현하는 실습 코드입니다.

## 6. Local 실행과 Metric 확인

---

Project Root에서 Application을 실행합니다.

```bash
# [DEV: Terminal 1]
chmod +x gradlew
./gradlew bootRun
```

다른 Terminal에서 요청과 Actuator 응답을 확인합니다.

```bash
# [DEV: Terminal 2]
curl --fail http://127.0.0.1:8080/hello
curl --fail 'http://127.0.0.1:8080/process?millis=100'
curl --fail http://127.0.0.1:8080/actuator/health
curl --fail http://127.0.0.1:8080/actuator/prometheus
```

응답에서 다음 이름을 찾습니다. Micrometer의 점 구분 이름은 Prometheus 형식으로 변환되며 Timer의 단위는 초입니다.

| Application Meter | Prometheus에서 확인할 Metric |
| --- | --- |
| `app.visitations.total` | `app_visitations_total` |
| `app.process.latency` | `app_process_latency_seconds_count`, `app_process_latency_seconds_sum` |

`application="spring-boot-metrics-app"`과 Counter의 `region="region-a"` Label도 확인합니다. 실제 노출 이름은 사용하는 Registry Version의 Endpoint 응답을 기준으로 확인합니다.

`/hello`만 호출하면 방문 횟수만 증가합니다. `/process`를 호출하면 방문 횟수와 Timer Count·Sum이 함께 증가합니다. 잘못된 `millis`는 HTTP 400을 반환해야 합니다.

## 7. Image Build와 Registry Push

---

Project Root에 `Dockerfile`을 작성합니다.

```dockerfile
FROM eclipse-temurin:17-jdk-alpine AS build
WORKDIR /app
COPY gradlew .
COPY gradle gradle
COPY build.gradle settings.gradle ./
COPY src src
RUN chmod +x gradlew && ./gradlew --no-daemon bootJar -x test

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/build/libs/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build Stage는 JDK로 JAR를 생성하고 Runtime Stage는 JRE로 실행합니다. `-x test`는 실습 Build에서 Test를 생략하는 Option이며 검증 성공을 뜻하지 않습니다. 운영 Pipeline에서는 별도 Test를 통과시켜야 합니다.

Docker Hub에 `prometheus-springboot` Repository를 생성한 뒤 다음 변수의 Placeholder를 실제 값으로 바꿉니다.

```bash
# [BUILD]
METRICS_REGISTRY_USER="<CONTAINER_REGISTRY_USERNAME>"
METRICS_IMAGE_TAG="<TAG>"
METRICS_IMAGE="${METRICS_REGISTRY_USER}/prometheus-springboot:${METRICS_IMAGE_TAG}"

docker login
docker build -t "$METRICS_IMAGE" .
docker push "$METRICS_IMAGE"
```

Build Host와 Worker의 CPU Architecture가 다르면 Worker용 Image를 Build해야 합니다. Private Repository를 사용한다면 배포 Namespace에 `imagePullSecrets`를 별도로 구성합니다.

## 8. Kubernetes Application 배포

---

이미 `monitoring` Namespace에 `prometheus-stack` Release가 설치돼 있다고 가정합니다. Application과 ServiceMonitor도 같은 Namespace를 사용해 수집 범위를 명확히 합니다.

```bash
# [ADMIN]
helm list -n monitoring
kubectl get crd servicemonitors.monitoring.coreos.com
kubectl get prometheus -n monitoring
```

`spring-boot-metrics.yaml`을 작성합니다. Image Placeholder는 Push한 Image와 동일하게 변경합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-metrics-app
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-boot-metrics-app
  template:
    metadata:
      labels:
        app: spring-boot-metrics-app
    spec:
      containers:
        - name: spring-boot-metrics-app
          image: "<CONTAINER_REGISTRY_USERNAME>/prometheus-springboot:<TAG>"
          ports:
            - name: http
              containerPort: 8080
          resources:
            limits:
              cpu: "500m"
              memory: "512Mi"
            requests:
              cpu: "200m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-metrics-app-service
  namespace: monitoring
  labels:
    app: spring-boot-metrics-app
spec:
  selector:
    app: spring-boot-metrics-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
  type: ClusterIP
```

`requests`는 `resources` 안에 들어갑니다. Scheduler는 Request와 Node의 가용 Resource 등을 고려해 Pod를 배치하고, 선택된 Worker의 kubelet이 Container Runtime을 통해 Application을 실행합니다. Replica 2개가 반드시 서로 다른 Node에 배치되는 것은 아닙니다.

```bash
# [ADMIN]
kubectl apply -f spring-boot-metrics.yaml
kubectl rollout status deployment/spring-boot-metrics-app -n monitoring
kubectl get pods -n monitoring -l app=spring-boot-metrics-app -o wide
kubectl get service spring-boot-metrics-app-service -n monitoring
kubectl get endpointslices -n monitoring   -l kubernetes.io/service-name=spring-boot-metrics-app-service
```

EndpointSlice에서 실행 중인 Pod 주소와 Port `8080`을 확인합니다.

### Annotation을 사용하는 구성

다음 Pod Annotation은 해당 값을 해석하는 Scrape 설정이 Prometheus에 있을 때 활용할 수 있습니다.

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/path: "/actuator/prometheus"
  prometheus.io/port: "8080"
```

Annotation 자체는 Kubernetes나 Prometheus Operator가 수집을 보장하는 스위치가 아닙니다. 이번 배포는 ServiceMonitor로 수집하므로 이 Annotation을 추가할 필요가 없습니다. 두 방식이 모두 활성화되면 같은 Endpoint를 중복 수집할 수 있습니다.

## 9. ServiceMonitor로 수집 대상 지정

---

ServiceMonitor는 Service의 Metadata Label을 선택하고 그 Service의 이름 있는 Port를 참조합니다. Service가 다시 Pod Label로 Endpoint를 연결합니다.

| 연결 지점 | 일치해야 하는 값 |
| --- | --- |
| Prometheus → ServiceMonitor | Prometheus의 `serviceMonitorSelector`와 ServiceMonitor Metadata Label |
| ServiceMonitor → Service | `spec.selector.matchLabels`와 Service Metadata Label |
| Service → Pod | Service의 `spec.selector`와 Pod Label |
| Scrape Port | ServiceMonitor의 `endpoints.port`와 Service의 `ports.name` |

먼저 실제 Prometheus 설정을 확인합니다.

```bash
# [ADMIN]
kubectl get prometheus -n monitoring -o yaml
```

`spec.serviceMonitorSelector`가 `release: prometheus-stack`을 선택하는지, `serviceMonitorNamespaceSelector`가 `monitoring` Namespace를 포함하는지 확인합니다. Helm Release 이름만으로 Selector를 단정하지 않습니다. 설치 시 Values를 변경했다면 실제 Selector에 맞춰 아래 Label을 조정합니다.

`service-monitor.yml`을 작성합니다.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: spring-boot-metrics-monitor
  namespace: monitoring
  labels:
    release: prometheus-stack
spec:
  namespaceSelector:
    matchNames:
      - monitoring
  selector:
    matchLabels:
      app: spring-boot-metrics-app
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s
```

`port: http`는 숫자 `80`이나 Container Port `8080`이 아니라 Service Port 이름입니다. Operator는 API Server에 등록된 ServiceMonitor를 감지해 Scrape 설정을 조정하고, Prometheus는 Service의 Endpoint로 발견한 각 Pod에서 Metric을 수집합니다.

```bash
# [ADMIN]
kubectl apply -f service-monitor.yml
kubectl get servicemonitor spring-boot-metrics-monitor -n monitoring
kubectl get services -n monitoring -l app=spring-boot-metrics-app
```

## 10. 요청 생성과 Prometheus 확인

---

Application Service에 Port Forwarding을 실행합니다. 명령을 실행한 Host에서만 Local 주소에 접근할 수 있습니다.

```bash
# [ADMIN: Terminal 1]
kubectl port-forward service/spring-boot-metrics-app-service   8080:80 -n monitoring
```

다른 Terminal에서 요청을 반복합니다.

```bash
# [ADMIN: Terminal 2]
for request in 1 2 3 4 5; do
  curl --fail http://127.0.0.1:8080/hello
  curl --fail 'http://127.0.0.1:8080/process?millis=100'
done
curl --fail http://127.0.0.1:8080/actuator/prometheus
```

Service 대상 Port Forwarding은 선택된 Pod 하나에 연결됩니다. 두 Replica의 Counter가 동일하게 증가해야 하는 것은 아닙니다.

Prometheus Service 이름을 확인한 뒤 별도 Terminal에서 UI에 접근합니다.

```bash
# [ADMIN: Terminal 3]
kubectl get services -n monitoring
kubectl port-forward service/prometheus-stack-kube-prom-prometheus   9090:9090 -n monitoring
```

`http://127.0.0.1:9090`의 Targets 화면에서 `spring-boot-metrics-monitor` 관련 Target과 `UP` 상태를 확인합니다. UI Version에 따라 메뉴는 `Status → Targets` 또는 `Target health`로 표시될 수 있습니다.

15초 수집 주기 이후 Query 화면에서 다음 Metric을 확인합니다.

```promql
app_visitations_total{application="spring-boot-metrics-app"}
```

Timer는 다음 두 Metric을 각각 조회합니다.

```promql
app_process_latency_seconds_count{application="spring-boot-metrics-app"}
```

```promql
app_process_latency_seconds_sum{application="spring-boot-metrics-app"}
```

Pod별로 Label 조합이 달라 각각의 시계열이 나타납니다. 누적값은 Process 재시작 시 초기화될 수 있으므로 계속 유지되는 업무 원장으로 사용하지 않습니다.

## 11. Grafana Dashboard 생성

---

Grafana Service를 Local Port로 연결합니다.

```bash
# [ADMIN: 별도 Terminal]
kubectl port-forward service/prometheus-stack-grafana   3000:80 -n monitoring
```

Browser에서 `http://127.0.0.1:3000`에 접속해 다음 순서로 Panel을 만듭니다.

1. Dashboard 생성 메뉴에서 새 Dashboard와 Visualization을 추가합니다.

2. 설치된 Prometheus Data Source를 선택합니다.

3. 방문 횟수 Panel에 `app_visitations_total{application="spring-boot-metrics-app"}`을 입력합니다.

4. 처리 횟수 Panel에 `app_process_latency_seconds_count{application="spring-boot-metrics-app"}`을 입력합니다.

5. 총 처리 시간 Panel에 `app_process_latency_seconds_sum{application="spring-boot-metrics-app"}`을 입력하고 Unit을 seconds로 지정합니다.

6. 최근 15분 등 요청이 발생한 시간 범위를 선택하고 Dashboard를 저장합니다.

이 Panel들은 누적값 확인용입니다. 총 처리 시간은 평균 지연이나 p95가 아니며, p95 Panel을 만들려면 별도의 Histogram 또는 Percentile 설정이 필요합니다.

## 12. 수집되지 않을 때 점검 순서

---

| 증상 | 확인 대상 |
| --- | --- |
| Local Endpoint가 404 | Actuator·Prometheus Registry 의존성, Exposure 설정 |
| Endpoint가 401·403 | Spring Security의 Metric Endpoint 접근 정책 |
| Pod가 실행되지 않음 | Image 이름·Tag·Architecture, Registry 인증, Resource·Event |
| Target 자체가 없음 | Prometheus Selector, ServiceMonitor Namespace·Label, Service Label·Port 이름 |
| Target이 DOWN | Endpoint 경로, Application Listen Port, NetworkPolicy, 연결·인증 Error |
| Prometheus에는 값이 있으나 Grafana는 비어 있음 | Data Source, Query Label, 시간 범위 |

관리 Host에서 다음 명령으로 배포 상태와 Application Log를 확인합니다.

```bash
# [ADMIN]
kubectl describe deployment spring-boot-metrics-app -n monitoring
kubectl get events -n monitoring --sort-by=.metadata.creationTimestamp
kubectl logs -n monitoring -l app=spring-boot-metrics-app   --all-containers=true --tail=100 --prefix=true
kubectl get service spring-boot-metrics-app-service -n monitoring -o yaml
kubectl get servicemonitor spring-boot-metrics-monitor -n monitoring -o yaml
```

Pod가 Ready이고 Metric Endpoint가 응답하더라도 ServiceMonitor가 선택되지 않으면 수집되지 않습니다. Application의 노출 상태와 Prometheus의 수집 설정을 각각 확인합니다.

## 참고 자료

---

- [Spring Boot Metric과 Prometheus 연동](https://docs.spring.io/spring-boot/reference/actuator/metrics.html)

- [Spring Boot Actuator Endpoint](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html)

- [Micrometer Timer](https://docs.micrometer.io/micrometer/reference/concepts/timers.html)

- [Micrometer DistributionSummary](https://docs.micrometer.io/micrometer/reference/concepts/distribution-summaries.html)

- [Micrometer Histogram과 Percentile](https://docs.micrometer.io/micrometer/reference/concepts/histogram-quantiles.html)

- [Prometheus Operator 수집 문제 점검](https://prometheus-operator.dev/docs/platform/troubleshooting/)
