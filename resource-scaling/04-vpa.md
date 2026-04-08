# VPA — Request/Limit 자동 조정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- VPA의 세 컴포넌트(Recommender, Updater, Admission Controller)는 각각 무슨 역할인가?
- VPA Recommender는 어떤 과거 데이터를 분석해 권고값을 계산하는가?
- VPA Admission Controller가 파드 스펙을 수정하는 시점은 언제인가?
- HPA와 VPA를 동시에 사용하면 왜 충돌이 발생하는가?
- VPA의 `updateMode`별 동작 차이는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Request와 Limit을 처음 설정할 때 "적절한 값"을 알기 어렵다. 너무 낮게 설정하면 Throttling이나 OOM이 발생하고, 너무 높게 설정하면 자원이 낭비되어 불필요한 비용이 든다. VPA는 실제 사용 이력을 분석해 최적의 Request/Limit을 자동으로 계산해준다. 초기 설정의 불확실성을 줄이는 강력한 도구다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: VPA + HPA를 동시에 같은 Deployment에 설정
      → 파드가 계속 재시작됨

원리를 모를 때의 판단:
  "VPA가 최적화해주고, HPA가 개수를 맞춰주면 완벽하지 않나?"
  → 두 개 동시 설정

실제 충돌:
  HPA: CPU 사용률 60% → 파드 2개 → 3개로 스케일아웃
  VPA Updater: 새 권고값으로 파드 재생성 (스케일아웃 된 파드 종료)
  HPA: 다시 CPU 증가 → 스케일아웃 시도
  → VPA-HPA 충돌로 파드 불안정

올바른 사용:
  CPU 기반 HPA + VPA: 충돌 (비권장)
  커스텀 메트릭 HPA + VPA CPU 권고 (readonly): 가능
  VPA만 사용 (스케일링 없음): 권고값 적용
```

---

## ✨ 올바른 접근 (After — VPA 모드 이해)

```
VPA updateMode 선택:
  "Off"      → 권고값만 계산, 적용 안 함 (사람이 참고용)
  "Initial"  → 파드 생성 시에만 적용 (재시작 없음)
  "Auto"     → 권고값 벗어나면 파드 재시작해 적용 (기본값)
  "Recreate" → Auto와 동일

사용 전략:
  1단계: updateMode: Off → 권고값 수집 (2주)
  2단계: 권고값으로 수동 Request 업데이트
  3단계: updateMode: Initial → 신규 파드에만 적용
  4단계: 충분한 검증 후 updateMode: Auto 고려

HPA와 공존 방법:
  VPA: CPU/메모리 Request 최적화 (updateMode: Off or Initial)
  HPA: RPS나 커스텀 메트릭 기반 파드 수 조정
  → CPU 사용률 기반 HPA는 VPA와 충돌하므로 제외
```

---

## 🔬 내부 동작 원리

### VPA 세 컴포넌트

```
VPA 아키텍처:

┌─────────────────┐   HistogramCheckpoint      ┌──────────────────┐
│  VPA Recommender│ ─────────────────────────► │ VPA Admission    │
│                 │                            │ Controller       │
│  과거 사용량 분석   │                            │ (MutatingWebhook)│
│  권고값 계산       │                            │                  │
│  VPA 오브젝트     │                             │ 파드 생성 시       │
│  status 업데이트  │                             │ spec 수정         │
└────────┬────────┘                             └──────────────────┘
         │ recommended Request/Limit
         ▼
┌─────────────────┐
│  VPA Updater    │
│                 │
│ Auto 모드 시      │
│ 권고값 범위 벗어난   │
│ 파드 종료 (Evict) │
│ → 재생성 시 AC가   │
│   새 값으로 설정   │
└─────────────────┘
```

**Recommender — 권고값 계산**

```
수집 데이터:
  metrics-server의 CPU/메모리 사용량 (15초 주기)
  과거 8일~1주일치 이력 유지

권고값 알고리즘:
  CPU Request:
    95th percentile 사용량 + 여유분
    → 극단적 스파이크 제외한 안정적 값

  Memory Request:
    90th percentile 사용량
    → OOM 방지를 위해 여유있게

  Limit:
    Request × LimitRequestRatio (기본값: 없음 → Request와 동일)

예시:
  파드 CPU 사용 이력: [50m, 80m, 120m, 90m, 200m, 60m, ...]
  95th percentile ≈ 150m
  → CPU Request 권고: 150m (현재 설정 200m에서 낮춤 가능)
```

**Admission Controller — 파드 생성 시 스펙 수정**

```
파드 생성 요청 → Mutating Webhook
  → VPA Admission Controller

  1. 해당 파드가 VPA 대상인지 확인
     (VPA 오브젝트의 targetRef 매칭)
  
  2. VPA 오브젝트의 status.recommendation 조회
  
  3. 파드 스펙 수정:
     containers[].resources.requests.cpu    = recommended CPU
     containers[].resources.requests.memory = recommended Memory
     containers[].resources.limits.cpu      = recommended CPU Limit
     
  4. 수정된 스펙으로 파드 생성
```

**Updater — 권고값 벗어난 파드 강제 교체**

```
updateMode: Auto인 경우:

  VPA Updater가 주기적으로 체크:
    현재 파드 Request vs 권고 Request 비교
    
    Request < 권고 * LowerBound → 파드 자원 부족 → Evict
    Request > 권고 * UpperBound → 파드 자원 낭비 → Evict
  
  Evict된 파드:
    Deployment/ReplicaSet이 재생성
    재생성 시 Admission Controller가 새 권고값 적용

  단점:
    파드 재시작 = 서비스 중단 가능
    → 반드시 replicas >= 2 이상 + PDB 설정 권장
```

### VPA 오브젝트 구조

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"   # Off / Initial / Auto / Recreate
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 50m         # 권고값 하한
        memory: 64Mi
      maxAllowed:
        cpu: 2           # 권고값 상한
        memory: 2Gi
      controlledResources:
      - cpu
      - memory
```

---

## 💻 실전 실험

### 1. VPA 설치 (Kind)

```bash
# VPA 설치
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# VPA 컴포넌트 확인
kubectl get pods -n kube-system | grep vpa
# vpa-admission-controller-xxx   1/1  Running
# vpa-recommender-xxx            1/1  Running
# vpa-updater-xxx                1/1  Running
```

### 2. VPA 설정 및 권고값 확인

```bash
# 테스트 Deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpa-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vpa-test
  template:
    metadata:
      labels:
        app: vpa-test
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: 500m    # 의도적으로 과도하게 설정
            memory: 512Mi
EOF

# VPA 오브젝트 생성 (Off 모드 - 권고만)
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-test-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-test
  updatePolicy:
    updateMode: "Off"
EOF

# 권고값 확인 (수분 후)
kubectl describe vpa vpa-test-vpa | grep -A20 "Recommendation:"
# Recommendation:
#   Container Recommendations:
#     Container Name: app
#     Lower Bound:
#       Cpu:     25m
#       Memory:  131072k
#     Target:
#       Cpu:     50m        ← 현재 500m에서 대폭 낮춤 권고
#       Memory:  155Mi      ← 현재 512Mi에서 낮춤 권고
#     Upper Bound:
#       Cpu:     200m
#       Memory:  512Mi
```

### 3. VPA 권고값으로 수동 업데이트

```bash
# Off 모드에서 권고값 확인 후 수동 적용
kubectl set resources deployment vpa-test \
  --requests=cpu=50m,memory=155Mi \
  --limits=cpu=200m,memory=512Mi

# 파드 재시작 확인
kubectl rollout status deployment vpa-test
```

### 4. HPA + VPA 충돌 시연

```bash
# CPU HPA 설정
kubectl autoscale deployment vpa-test --cpu-percent=50 --min=2 --max=5

# VPA Auto 모드 설정 (동시)
kubectl patch vpa vpa-test-vpa --type=merge \
  -p='{"spec":{"updatePolicy":{"updateMode":"Auto"}}}'

# 잠시 후 파드 불안정 확인 (재시작 반복)
kubectl get pods -l app=vpa-test -w
# RESTARTS가 계속 증가하는 현상 관찰
```

---

## 📊 VPA updateMode별 비교

| 모드 | 파드 재시작 | 적용 시점 | 주요 용도 |
|-----|---------|---------|---------|
| Off | ❌ | 없음 | 권고값 참고용 |
| Initial | ❌ (신규만) | 파드 생성 시 | 새 파드에만 적용 |
| Auto | ✅ | 권고 범위 벗어날 때 | 자동 최적화 |
| Recreate | ✅ | Auto와 동일 | Auto의 명시적 표현 |

---

## ⚖️ 트레이드오프

**Auto 모드의 서비스 중단 위험**

VPA Updater가 파드를 Evict하면 순간적인 서비스 중단이 발생할 수 있다. Deployment replicas가 1이면 Evict 순간 파드가 0개가 된다. PodDisruptionBudget으로 최소 가용 파드 수를 보장하고, replicas를 2 이상으로 유지해야 한다. 단일 파드 서비스에는 Auto 모드보다 Initial이나 Off + 수동 적용이 안전하다.

**VPA 권고값의 과거 의존성**

Recommender는 과거 사용량 이력을 기반으로 권고한다. 새로 배포한 서비스나 트래픽 패턴이 크게 바뀐 경우에는 권고값이 부정확할 수 있다. 최소 수일~수주의 이력이 쌓인 후 권고값을 신뢰하는 것이 좋다.

---

## 📌 핵심 정리

```
VPA 세 컴포넌트:
  Recommender: 과거 사용량 분석 → Request/Limit 권고값 계산
  Admission Controller: 파드 생성 시 스펙 수정 (Mutating Webhook)
  Updater: Auto 모드 시 권고 범위 벗어난 파드 Evict → 재생성

updateMode:
  Off      → 권고만, 적용 없음 (1단계: 데이터 수집)
  Initial  → 신규 파드에만 (안전한 적용)
  Auto     → 자동 Evict + 재생성 (위험, PDB 필수)

HPA + VPA 충돌:
  CPU HPA + VPA Auto → 심각한 충돌 (비권장)
  커스텀 메트릭 HPA + VPA Off/Initial → 공존 가능

VPA 활용 전략:
  Off로 2주 데이터 수집 → 권고값 수동 반영 → 비용 최적화
```

---

## 🤔 생각해볼 문제

**Q1.** VPA `resourcePolicy.containerPolicies[].minAllowed`를 설정하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

Recommender가 너무 낮은 값을 권고하는 것을 방지한다. 예를 들어 트래픽이 적은 밤 시간 데이터로 CPU Request: 5m을 권고할 수 있는데, 이 값으로 설정되면 낮 시간 트래픽 급증 시 Throttling이 발생한다. `minAllowed.cpu: 50m`을 설정하면 권고값이 최소 50m 이상이 된다. 유사하게 `maxAllowed`는 권고값의 상한을 제한해, VPA가 지나치게 많은 자원을 할당하는 것을 방지한다.

</details>

**Q2.** VPA는 Init Container의 Request도 자동 조정하는가?

<details>
<summary>해설 보기</summary>

그렇다. VPA는 Init Container를 포함한 모든 컨테이너를 대상으로 한다. `resourcePolicy.containerPolicies[].containerName: '*'`으로 설정하면 Init Container와 메인 컨테이너 모두에 권고값이 적용된다. 특정 컨테이너만 대상으로 하려면 containerName을 명시한다. Init Container는 순차 실행이므로 메인 컨테이너와 다른 패턴을 가질 수 있고, 개별적으로 권고값이 계산된다.

</details>

**Q3.** VPA Off 모드로 권고값을 수집한 후, 수동으로 Deployment의 Request를 변경할 때 권고값과 크게 차이나면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

권고값을 참고해 점진적으로 조정하는 것이 좋다. 권고 CPU가 현재 500m → 50m을 권고한다면, 한 번에 50m으로 낮추는 것은 위험하다. 이력이 부정확하거나 예외적인 상황이 반영되지 않았을 수 있다. 중간값(200m)으로 낮추고 모니터링 후 추가 조정하는 단계적 접근이 안전하다. CPU Throttling(`container_cpu_cfs_throttled_seconds_total`) 메트릭과 메모리 OOM 이벤트를 모니터링하면서 Request를 낮춰야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: HPA — 메트릭 파이프라인과 스케일 알고리즘](./03-hpa.md)** | **[홈으로 🏠](../README.md)** | **[다음: 클러스터 오토스케일러 — 노드 추가와 스케일다운 ➡️](./05-cluster-autoscaler.md)**

</div>
