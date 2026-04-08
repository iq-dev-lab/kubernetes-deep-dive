# HPA — 메트릭 파이프라인과 스케일 알고리즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HPA가 메트릭을 수집하는 전체 파이프라인은 어떻게 구성되는가?
- `desiredReplicas = ceil(currentReplicas × currentMetric / desiredMetric)` 공식은 어떻게 적용되는가?
- 스케일다운에 5분 쿨다운이 필요한 이유는 무엇인가?
- Prometheus 메트릭으로 HPA를 구성하려면 무엇이 추가로 필요한가?
- HPA가 동작하지 않을 때 어떻게 진단하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

트래픽이 급증할 때 수동으로 `kubectl scale`을 실행하는 것은 현실적이지 않다. HPA는 메트릭을 기반으로 파드 수를 자동으로 조정한다. 그러나 HPA가 동작하지 않거나 너무 민감하게 반응해 파드가 계속 생성·삭제되는 "flapping" 현상은 HPA 내부를 이해해야 해결할 수 있다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: HPA를 설정했는데 CPU 70%임에도 스케일아웃 안 됨

원리를 모를 때의 판단:
  "HPA가 고장났나?" → kubectl delete hpa → 재생성
  → 동일 증상

실제 원인:
  metrics-server가 설치 안 됨
  → HPA Controller가 메트릭 조회 불가
  → kubectl get hpa 시 TARGETS 컬럼에 <unknown>

진단 순서:
  1. kubectl top pods → 결과 없음 = metrics-server 문제
  2. kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
     → status.conditions Available: False
  3. kubectl logs -n kube-system deployment/metrics-server
  4. metrics-server 설치 후 → HPA 정상 동작
```

---

## ✨ 올바른 접근 (After — HPA 파이프라인을 이해한 설계)

```
HPA 동작 전 필수 확인:
  kubectl top pods → 메트릭이 보이는가?
  kubectl get hpa → TARGETS가 <unknown>이 아닌가?
  kubectl describe hpa → Events에 오류 없는가?

HPA 설계 원칙:
  CPU 기반: CPU Request 반드시 설정 (Request 기준 비율)
  메모리 기반: 일반적으로 비권장 (메모리는 즉시 해제 안 됨)
  커스텀 메트릭: RPS, 큐 길이 등 비즈니스 메트릭 선호
  minReplicas: 0으로 설정하면 트래픽 없을 때 파드 0개 (KEDA 사용 권장)
```

---

## 🔬 내부 동작 원리

### 메트릭 수집 파이프라인

```
파드 내 프로세스 (CPU/메모리 사용)
      │
      ▼
kubelet Summary API
  /api/v1/nodes/<node>/proxy/stats/summary
  각 파드의 CPU/메모리 사용량 노출

      │
      ▼
Metrics Server (Deployment)
  모든 노드의 kubelet에서 메트릭 수집 (15초 주기)
  메모리 캐시 유지

      │ API 집계 레이어 (APIService)
      ▼
metrics.k8s.io API
  kubectl top pods / top nodes 의 실제 소스
  GET /apis/metrics.k8s.io/v1beta1/pods

      │
      ▼
HPA Controller (Controller Manager 내)
  15초마다 metrics.k8s.io 조회
  목표 메트릭 대비 현재 값 계산
  desiredReplicas 계산 → Deployment 스케일 변경
```

**Prometheus Adapter를 통한 커스텀 메트릭**

```
애플리케이션 → Prometheus 메트릭 노출 (/metrics)
      │
      ▼
Prometheus Server (수집)

      │
      ▼
Prometheus Adapter (Deployment)
  custom.metrics.k8s.io API 구현
  Prometheus 쿼리 결과를 k8s 메트릭 API 형식으로 변환

      │
      ▼
HPA Controller
  custom.metrics.k8s.io 조회
  예: http_requests_per_second 기반 스케일링
```

### HPA 스케일 알고리즘

```
기본 공식:
  desiredReplicas = ceil(currentReplicas × (currentMetric / desiredMetric))

예시 (CPU 기반):
  현재 파드: 2개
  각 파드 CPU 사용: 80% (평균)
  HPA 목표: 50%

  desiredReplicas = ceil(2 × (80 / 50))
                 = ceil(2 × 1.6)
                 = ceil(3.2)
                 = 4 → 4개로 스케일아웃

  스케일아웃 후:
    4개 파드로 80% 부하 → 각 파드 40%
    다음 평가: ceil(4 × (40/50)) = ceil(3.2) = 4 → 안정
```

**여러 파드의 메트릭 평균**

```
3개 파드 CPU: [90%, 85%, 70%]
평균: (90+85+70)/3 = 81.67%

desiredReplicas = ceil(3 × 81.67/50) = ceil(4.9) = 5
```

**CPU Request 기준 비율 계산**

```
CPU Utilization (percentage)는 Request 대비 비율:
  파드 CPU Request: 200m
  현재 CPU 사용: 160m
  Utilization: 160/200 × 100 = 80%

  → Request가 설정되지 않으면 HPA CPU 메트릭 사용 불가
```

### 스케일다운 안정화

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # 기본 5분
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60    # 1분마다 최대 10% 파드 감소
    scaleUp:
      stabilizationWindowSeconds: 0    # 스케일아웃은 즉시
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15   # 15초마다 최대 100% 파드 증가
```

```
스케일다운 안정화 (stabilizationWindowSeconds: 300) 이유:
  메트릭이 순간적으로 낮아졌다가 다시 높아지는 "flapping" 방지
  
  예시 없이:
    t=0: CPU 80% → 4개로 스케일아웃
    t=15s: CPU 40% → 2개로 스케일다운 시도
    t=30s: CPU 75% → 4개로 다시 스케일아웃
    → 계속 반복 (파드 생성/삭제 비용 발생)
  
  stabilizationWindow 300초 적용:
    최근 5분간의 최댓값을 기준으로 스케일다운 여부 결정
    t=0 ~ t=300s 동안 최대 4가 권고됐으면 → 4 유지
    t=300s 이후에도 낮으면 그때 스케일다운
```

---

## 💻 실전 실험

### 1. metrics-server 설치 및 확인 (Kind)

```bash
# Kind 클러스터에 metrics-server 설치
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Kind에서는 --kubelet-insecure-tls 필요
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}
]'

# 메트릭 수집 확인 (1~2분 소요)
kubectl top pods
# NAME       CPU(cores)   MEMORY(bytes)
# nginx-xxx  1m           3Mi
```

### 2. CPU 기반 HPA 설정

```bash
# 테스트용 Deployment
kubectl create deployment hpa-test --image=nginx --replicas=1
kubectl set resources deployment hpa-test --requests=cpu=100m --limits=cpu=200m

# HPA 생성
kubectl autoscale deployment hpa-test \
  --cpu-percent=50 \
  --min=1 \
  --max=5

# HPA 상태 확인
kubectl get hpa hpa-test
# NAME       REFERENCE         TARGETS   MINPODS  MAXPODS  REPLICAS
# hpa-test   Deployment/hpa-test  5%/50%     1        5        1
```

### 3. 부하 생성 후 스케일아웃 관찰

```bash
# CPU 부하 생성
kubectl run load-gen --image=busybox --rm -it -- \
  sh -c "while true; do wget -qO- http://hpa-test; done" &

# HPA 변화 관찰
kubectl get hpa hpa-test -w
# NAME       TARGETS      REPLICAS
# hpa-test   5%/50%       1
# hpa-test   85%/50%      1  ← 부하 증가
# hpa-test   85%/50%      2  ← 스케일아웃 시작
# hpa-test   60%/50%      3
# hpa-test   45%/50%      4  ← 목표 근접
```

### 4. HPA 이벤트로 스케일링 이력 확인

```bash
kubectl describe hpa hpa-test | tail -20
# Events:
#   Normal  SuccessfulRescale  2m  horizontal-pod-autoscaler
#     New size: 3; reason: cpu resource utilization (percentage of request) above target
#   Normal  SuccessfulRescale  1m  horizontal-pod-autoscaler
#     New size: 4; reason: cpu resource utilization (percentage of request) above target
```

### 5. v2 HPA API로 동작 세밀 조정

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: advanced-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60    # 1분에 최대 4개 추가
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 25
        periodSeconds: 60    # 1분에 최대 25% 감소
```

---

## 📊 HPA 메트릭 타입 비교

| 메트릭 타입 | API | 예시 | 요구사항 |
|----------|-----|-----|---------|
| CPU Utilization | metrics.k8s.io | targetCPUUtilizationPercentage | metrics-server, CPU Request 필수 |
| 메모리 Utilization | metrics.k8s.io | memory 사용량 | metrics-server (비권장) |
| 커스텀 메트릭 | custom.metrics.k8s.io | RPS, QPS | Prometheus + Adapter |
| 외부 메트릭 | external.metrics.k8s.io | SQS 큐 길이 | External Metrics Adapter |

---

## ⚖️ 트레이드오프

**CPU 기반 vs 커스텀 메트릭 기반**

CPU 기반 HPA는 설정이 단순하지만, I/O 집약적 서비스(DB 쿼리 대기, 외부 API 호출)에서는 CPU가 낮아도 응답 지연이 발생하는 경우 스케일아웃이 트리거되지 않는다. RPS(초당 요청 수)나 응답 지연 기반 커스텀 메트릭이 더 실질적인 지표가 될 수 있다. 단, Prometheus + Adapter 설정이 추가로 필요하다.

**minReplicas: 0의 위험성**

HPA minReplicas를 0으로 설정하면 트래픽이 없을 때 파드가 0개가 되어 비용을 절감할 수 있다. 그러나 첫 요청이 들어올 때 파드가 0개이므로 콜드 스타트 지연이 발생한다. Spring Boot처럼 기동이 느린 애플리케이션에서는 수십 초의 지연이 생긴다. KEDA(Kubernetes Event-driven Autoscaling)의 ScaledObject를 사용하면 이 시나리오를 더 정교하게 처리할 수 있다.

---

## 📌 핵심 정리

```
HPA 메트릭 파이프라인:
  kubelet → metrics-server → metrics.k8s.io → HPA Controller
  Prometheus → Adapter → custom.metrics.k8s.io → HPA Controller

스케일 공식:
  desiredReplicas = ceil(현재 파드 수 × 현재값 / 목표값)

안정화 (flapping 방지):
  scaleDown: 5분 안정화 윈도우 (기본)
  scaleUp: 즉시 반응 (기본)

HPA 동작 안 할 때 진단:
  kubectl top pods  → metrics-server 확인
  kubectl get hpa   → TARGETS <unknown> → 메트릭 없음
  kubectl describe hpa → Events 오류 확인

CPU 기반 HPA 전제조건:
  반드시 CPU Request 설정 (Request 대비 비율로 계산)
```

---

## 🤔 생각해볼 문제

**Q1.** HPA가 파드 수를 2 → 1로 스케일다운하려 하는데, PodDisruptionBudget이 minAvailable: 2로 설정되어 있다. 어떻게 되는가?

<details>
<summary>해설 보기</summary>

HPA가 스케일다운 시도 시 Deployment Controller가 파드를 삭제하려 한다. PodDisruptionBudget이 minAvailable: 2를 요구하므로, 현재 2개 파드에서 1개를 삭제하면 minAvailable 조건을 위반한다. PDB가 삭제를 차단하고, HPA는 파드를 줄이지 못한다. 1개로 줄이려면 PDB를 minAvailable: 1로 수정하거나 제거해야 한다. PDB는 HPA 스케일다운에도 영향을 미치므로, 너무 엄격한 PDB 설정은 의도치 않게 스케일다운을 막을 수 있다.

</details>

**Q2.** metrics-server는 몇 초 주기로 메트릭을 수집하는가? HPA는 몇 초 주기로 평가하는가?

<details>
<summary>해설 보기</summary>

metrics-server는 기본 15초 주기로 kubelet에서 메트릭을 수집한다(`--metric-resolution` 플래그). HPA Controller는 기본 15초 주기로 메트릭을 평가한다(`--horizontal-pod-autoscaler-sync-period`). 따라서 실제 CPU 사용량 변화가 HPA 스케일링 결정으로 반영되는 데는 최대 15+15=30초가 걸릴 수 있다. 더 빠른 반응이 필요하면 두 값을 줄일 수 있지만, API Server 부하가 증가한다.

</details>

**Q3.** Deployment와 HPA를 동시에 운영할 때, `kubectl scale deployment`로 수동으로 파드 수를 바꾸면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

HPA가 다음 평가 주기(15초)에 Deployment의 replicas를 다시 계산된 값으로 덮어쓴다. 예를 들어 HPA가 3개로 계산했는데 수동으로 1개로 줄이면, 15초 안에 HPA가 다시 3개로 복원한다. HPA와 함께 쓸 때는 수동 `kubectl scale`이 의미 없다. 단, minReplicas/maxReplicas 범위를 벗어나도 HPA가 범위 내로 강제 조정한다. 수동으로 파드 수를 유지하려면 HPA를 삭제하거나 비활성화해야 한다.

</details>

---

> ⬅️ 이전: [02. QoS 클래스 — OOM 발생 시 파드 종료 순서](./02-qos-classes.md)  
> ➡️ 다음: [04. VPA — Request/Limit 자동 조정](./04-vpa.md)
