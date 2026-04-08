# 모니터링과 로깅 — Prometheus와 Loki 파이프라인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Prometheus Operator의 ServiceMonitor는 어떻게 동적으로 스크랩 대상을 관리하는가?
- `kubectl top pod`의 메트릭이 Metrics Server → API 집계 → HPA Controller로 연결되는 경로는?
- Loki가 파드 레이블을 기반으로 로그를 수집하는 방식은?
- 로그 레벨과 로그 보존 기간의 비용 트레이드오프는?
- 메트릭 기반 알람(Alert)이 로그 기반 알람보다 빠른 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"장애가 발생했는데 로그가 없다"거나 "메트릭은 있는데 무엇이 문제인지 모르겠다"는 상황은 모니터링/로깅 파이프라인을 이해하지 못한 경우가 많다. Prometheus, Loki, Grafana는 각각 독립적이지만 연동되어 동작한다. 파이프라인 전체를 이해해야 어디서 데이터가 수집되고, 어디서 보여지며, 어디서 끊기는지 알 수 있다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 새 Service를 추가했는데 Prometheus에 메트릭이 안 보임

원리를 모를 때의 판단:
  "Prometheus를 재시작하면 되겠지"
  → 재시작 → 동일 증상

실제 원인:
  Prometheus Operator 사용 시:
    ServiceMonitor 리소스를 생성해야 함
    ServiceMonitor.spec.selector가 Service 레이블과 매칭해야 함
    ServiceMonitor가 Prometheus CRD의 serviceMonitorSelector와 매칭해야 함

  직접 설치 Prometheus 사용 시:
    prometheus.yml의 scrape_configs에 수동 추가 필요

진단:
  kubectl get servicemonitor -A → 생성됐는가?
  kubectl get service -l <labels> → 레이블 매칭 확인
  Prometheus UI → Status → Targets → 스크랩 상태 확인
```

---

## ✨ 올바른 접근 (After — 파이프라인을 이해한 설계)

```
모니터링 스택 선택:

  메트릭 수집: Prometheus (풀 방식)
    → 각 파드/서비스가 /metrics 엔드포인트 노출
    → Prometheus가 주기적으로 수집(scrape)

  메트릭 저장: Prometheus TSDB (또는 Thanos, VictoriaMetrics for 대규모)

  시각화: Grafana

  알람: Alertmanager (Prometheus 룰 기반)

  로그 수집: Promtail or Fluentd (각 노드에 DaemonSet)
    → 파드 로그 파일(/var/log/pods/)을 tail

  로그 저장: Loki (레이블 기반 인덱싱)

  로그 조회: Grafana (LogQL)
```

---

## 🔬 내부 동작 원리

### Prometheus Operator 동작 원리

```
Prometheus Operator 설치:
  CRD 등록: Prometheus, ServiceMonitor, PodMonitor, AlertmanagerConfig 등
  Controller: CRD Watch → Prometheus 설정 자동 생성

ServiceMonitor로 스크랩 대상 동적 관리:
  
  Service 레이블:
    app: my-api
    
  ServiceMonitor:
    spec:
      selector:
        matchLabels:
          app: my-api         # 이 레이블의 Service 자동 감지
      endpoints:
      - port: metrics
        interval: 30s
        path: /metrics
      namespaceSelector:
        matchNames: ["production"]
  
  Prometheus CR:
    spec:
      serviceMonitorSelector:
        matchLabels:
          release: prometheus   # 이 레이블의 ServiceMonitor 포함

  동작 흐름:
    1. ServiceMonitor 생성
    2. Operator가 감지 → prometheus.yml scrape_config 생성
    3. Prometheus 재로드 (config reloader sidecar)
    4. Prometheus가 Service → Endpoints → 파드 IP:port에 HTTP GET /metrics
    5. 메트릭 수집 완료 → TSDB 저장
```

### Prometheus 메트릭 형식

```
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200",path="/api/users"} 4521
http_requests_total{method="POST",status="500",path="/api/order"} 3

# TYPE response_latency_seconds histogram
response_latency_seconds_bucket{le="0.1"} 1200
response_latency_seconds_bucket{le="0.5"} 4800
response_latency_seconds_bucket{le="1.0"} 5100
response_latency_seconds_sum 2400.5
response_latency_seconds_count 5100

# TYPE memory_usage_bytes gauge
memory_usage_bytes{pod="my-api-abc"} 268435456
```

### kubectl top → HPA 메트릭 파이프라인

```
파드 내 프로세스 (CPU/메모리 사용)
  ↓
kubelet Summary API
  /api/v1/nodes/<node>/proxy/stats/summary
  (각 파드의 CPU 사용, 메모리 사용 집계)
  ↓
Metrics Server (Deployment, 클러스터당 1개)
  모든 노드 kubelet에서 15초마다 수집
  메모리에 최신 값만 보관 (시계열 아님)
  ↓
metrics.k8s.io API (APIService로 등록)
  kubectl top pods → 여기 조회
  kubectl top nodes → 여기 조회
  ↓
HPA Controller (Controller Manager 내)
  15초마다 metrics.k8s.io 조회
  → desiredReplicas 계산 → Deployment 스케일 조정
```

### Loki 로그 파이프라인

```
파드 컨테이너 stdout/stderr
  ↓
kubelet이 노드 로그 파일로 저장
  /var/log/pods/<namespace>_<pod>_<uid>/<container>/*.log
  /var/log/containers/ → 위 경로의 심링크
  ↓
Promtail (DaemonSet, 각 노드에 1개)
  /var/log/pods/ 디렉토리 tail
  파드 레이블 자동 추가 (namespace, pod, container, app 등)
  ↓
Loki (로그 저장)
  레이블 기반 인덱싱 (Prometheus와 동일한 레이블 개념)
  로그 본문은 인덱싱 안 함 (검색 시 grep)
  ↓
Grafana Explore (LogQL 쿼리)
  {namespace="production", app="my-api"} |= "ERROR"
```

```
LogQL 예시:
  # production ns의 my-api에서 ERROR 로그만
  {namespace="production", app="my-api"} |= "ERROR"

  # 최근 1시간 동안 500 에러 수
  count_over_time({app="my-api"} |= "status=500" [1h])

  # 응답 시간 추출 (structured log)
  {app="my-api"} | json | latency_ms > 1000
```

### Alertmanager 알람 파이프라인

```
PrometheusRule CRD:
  apiVersion: monitoring.coreos.com/v1
  kind: PrometheusRule
  spec:
    groups:
    - name: my-api.rules
      rules:
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m])
          / rate(http_requests_total[5m]) > 0.05
        for: 5m           # 5분 지속 시 알람
        labels:
          severity: critical
        annotations:
          summary: "5xx 에러율 5% 초과"

Alertmanager 라우팅:
  routes:
  - match:
      severity: critical
    receiver: slack-critical    # Slack 채널
  - match:
      severity: warning
    receiver: email-team        # 이메일
```

---

## 💻 실전 실험

### 1. kube-prometheus-stack 설치 (Helm)

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm install kube-prom-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=admin123

kubectl get pods -n monitoring
# prometheus-kube-prom-stack-prometheus-0  ← Prometheus
# alertmanager-kube-prom-stack-alertmanager-0  ← Alertmanager
# kube-prom-stack-grafana-xxx  ← Grafana
```

### 2. ServiceMonitor 생성

```bash
# 메트릭 노출 앱 배포
kubectl create deployment metrics-app --image=nginx --port=9113
kubectl expose deployment metrics-app --port=9113

# ServiceMonitor 생성
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: metrics-app
  namespace: default
  labels:
    release: kube-prom-stack   # Prometheus의 serviceMonitorSelector와 매칭
spec:
  selector:
    matchLabels:
      app: metrics-app
  endpoints:
  - port: "9113"
    interval: 30s
    path: /metrics
EOF

# Prometheus UI에서 Target 확인
kubectl port-forward -n monitoring svc/kube-prom-stack-prometheus 9090:9090 &
# http://localhost:9090/targets
```

### 3. kubectl top 확인

```bash
# Metrics Server 필요
kubectl top pods -A
# NAMESPACE   NAME              CPU(cores)   MEMORY(bytes)
# monitoring  prometheus-xxx    45m          512Mi
# default     my-app-xxx        5m           64Mi

# 정렬
kubectl top pods -A --sort-by=memory | head -10
```

### 4. Loki 스택 설치 및 로그 조회

```bash
helm repo add grafana https://grafana.github.io/helm-charts

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set grafana.enabled=false  # 기존 Grafana 사용

# Grafana에서 Loki 데이터소스 추가 후 LogQL 실행
# {namespace="default"} |= "error"
```

---

## 📊 로그 레벨별 비용 트레이드오프

| 로그 레벨 | 데이터 양 | 저장 비용 | 장애 진단 가치 | 운영 부하 |
|---------|---------|---------|------------|---------|
| ERROR only | 매우 적음 | 매우 낮음 | 높음 (노이즈 적음) | 낮음 |
| WARN+ERROR | 적음 | 낮음 | 높음 | 낮음 |
| INFO+WARN+ERROR | 중간 | 중간 | 중간 (대부분 커버) | 중간 |
| DEBUG (전체) | 매우 많음 | 높음 | 매우 높음 | 높음 |

---

## ⚖️ 트레이드오프

**메트릭 vs 로그 기반 알람**

메트릭 기반 알람(Prometheus)은 수집 주기(15~30초) 단위로 집계된 수치를 기반으로 빠르게 반응한다. 로그 기반 알람은 로그가 쌓이고, Promtail이 읽고, Loki에 저장되는 지연이 있어 보통 수십 초~수 분이 더 걸린다. 빠른 알람은 메트릭 기반으로, 상세 원인 분석은 로그로 하는 것이 일반적인 전략이다.

**Loki vs Elasticsearch**

Loki는 레이블만 인덱싱하고 로그 본문은 인덱싱하지 않아 저장 비용이 낮다. 대신 전문 검색(Full-text search)이 느릴 수 있다. Elasticsearch는 모든 내용을 인덱싱하여 빠른 검색을 제공하지만 저장 비용이 높다. 로그 양이 많고 비용이 중요하다면 Loki, 복잡한 검색 쿼리가 많으면 Elasticsearch가 유리하다.

---

## 📌 핵심 정리

```
Prometheus 수집 경로:
  서비스 /metrics → ServiceMonitor → Prometheus scrape
  Operator가 ServiceMonitor → prometheus.yml 자동 생성

kubectl top → HPA 파이프라인:
  kubelet → Metrics Server → metrics.k8s.io API
  → kubectl top (사람), HPA Controller (자동 스케일)

Loki 로그 파이프라인:
  파드 stdout → 노드 로그 파일 → Promtail(DaemonSet)
  → Loki 저장 → Grafana LogQL 조회
  레이블 기반 인덱싱 → 낮은 저장 비용

알람 전략:
  빠른 감지: Prometheus 메트릭 기반 AlertRule
  원인 분석: Loki 로그 기반 조회
  두 가지 연계: Grafana에서 메트릭+로그 통합 뷰
```

---

## 🤔 생각해볼 문제

**Q1.** Prometheus가 파드를 직접 스크랩하는 방식과, Pushgateway를 통해 메트릭을 받는 방식의 차이는?

<details>
<summary>해설 보기</summary>

Prometheus는 기본적으로 Pull 방식이다. Prometheus가 주기적으로 각 타겟의 `/metrics`를 HTTP GET으로 수집한다. 이 방식은 타겟이 살아있는지 확인(Up/Down 메트릭)이 가능하고, 수집 간격을 중앙에서 제어한다. Push 방식(Pushgateway)은 배치 잡처럼 짧게 실행하는 프로세스에 사용한다. 배치가 완료될 때 Pushgateway에 메트릭을 push하고, Prometheus는 Pushgateway를 스크랩한다. 단, Pushgateway는 죽은 배치의 메트릭을 계속 보유하므로 잘못된 알람을 유발할 수 있다.

</details>

**Q2.** 파드의 로그가 Loki에 보이지 않는다. 어떤 순서로 진단하는가?

<details>
<summary>해설 보기</summary>

(1) 파드가 로그를 stdout/stderr로 출력하는지 확인(`kubectl logs`로 로그 보임?). (2) 해당 노드에 Promtail 파드가 Running인지 확인(`kubectl get pods -n monitoring -l app=promtail`). (3) Promtail 로그에서 파드 로그 파일 tail 오류 없는지 확인(`kubectl logs -n monitoring promtail-xxx`). (4) Loki에 데이터가 도달하는지 확인(Loki 로그). (5) Grafana에서 레이블 필터 확인(올바른 namespace, pod 레이블 사용). (6) Promtail config의 relabeling 규칙 확인.

</details>

**Q3.** 동일 메트릭 이름으로 여러 파드가 메트릭을 노출할 때, Prometheus는 어떻게 구분하는가?

<details>
<summary>해설 보기</summary>

Prometheus는 스크랩 시 자동으로 `job`, `instance` 레이블을 추가한다. `instance`는 스크랩 대상의 `host:port`, `job`은 ServiceMonitor 이름이다. 또한 Prometheus Operator는 `pod`, `namespace`, `service` 같은 Kubernetes 레이블을 자동으로 추가한다. 따라서 `http_requests_total{pod="my-api-abc123", namespace="production"}` 형태로 각 파드를 구분한다. PromQL에서 `sum by (pod) (rate(http_requests_total[5m]))`으로 파드별 집계가 가능하다.

</details>

---

<div align="center">

**[⬅️ 이전: ConfigMap과 Secret — 설정 분리의 원리](./04-configmap-secret.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — Operator 패턴 ➡️](../advanced-patterns/01-operator-crd.md)**

</div>
