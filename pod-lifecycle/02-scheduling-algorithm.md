# 파드 스케줄링 알고리즘 — Filtering과 Scoring

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Filtering 단계는 어떤 기준으로 노드를 제거하고, Scoring 단계는 어떻게 점수를 매기는가?
- Taint와 Toleration은 어떻게 동작하고, NodeAffinity와 무엇이 다른가?
- `FailedScheduling` 이벤트의 메시지를 보고 어떤 조건이 문제인지 어떻게 읽는가?
- 동일한 애플리케이션 파드들이 서로 다른 노드에 분산되도록 강제하는 방법은?
- Scheduler가 특정 파드를 항상 같은 노드에 배치하게 하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"파드 3개를 만들었는데 모두 같은 노드에 몰렸다", "GPU 노드에 일반 파드가 올라가서 GPU 작업이 대기한다", "control-plane 노드에 워크로드가 올라간다" — 이 모든 상황이 스케줄링 설정 문제다. 스케줄링 알고리즘을 이해하면 파드가 어디에 올라갈지 예측하고, 원하는 배치 전략을 정확하게 설계할 수 있다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: HA 구성을 위해 Deployment replicas=3으로 설정했는데
      3개 파드가 모두 같은 노드에 배치됨
      해당 노드 장애 시 전체 서비스 중단

원리를 모를 때의 판단:
  "쿠버네티스가 자동으로 분산하지 않나?"
  → 그냥 방치
  → 노드 장애 발생 → 전체 다운

실제 동작:
  Scheduler의 기본 Scoring에는 분산 선호가 있지만
  노드 간 리소스 차이가 없으면 결과가 결정적이지 않음
  강제 분산을 보장하려면 PodAntiAffinity 설정 필요

올바른 설정:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: my-app
        topologyKey: kubernetes.io/hostname
  → 같은 hostname 노드에 같은 앱 파드가 2개 올라가지 않도록 강제
```

---

## ✨ 올바른 접근 (After — 스케줄링을 이해한 설계)

```
스케줄링 제어 도구 선택 기준:

"특정 노드에만 올리고 싶다"
  → nodeSelector (단순, 권장)
  → NodeAffinity (복잡한 조건, required/preferred 구분)

"특정 노드에 올리면 안 된다"
  → Taint + Toleration (노드 관리자 관점)
  → NodeAffinity NotIn (파드 관점)

"같은 앱 파드끼리 다른 노드에 분산"
  → PodAntiAffinity required (강제 분산)

"DB 파드와 가까운 노드에 앱 배치"
  → PodAffinity preferred (선호, 강제 아님)

"GPU 노드를 GPU 작업에만 독점"
  → GPU 노드에 Taint 추가 → GPU 파드에만 Toleration 부여
```

---

## 🔬 내부 동작 원리

### Scheduler 사이클 전체 구조

```
새 파드 감지 (Watch: nodeName="")
      │
      ▼
┌─────────────────────────────────────────┐
│  Phase 1: Filtering (부적합 노드 제거)   │
│                                          │
│  모든 노드를 대상으로 아래 조건 순차 검사: │
│  ① PodFitsResources: CPU/메모리 충분?   │
│  ② PodFitsHost: nodeName 지정 시 일치?  │
│  ③ PodFitsHostPorts: 포트 충돌 없음?    │
│  ④ NoDiskConflict: 볼륨 충돌 없음?     │
│  ⑤ NoVolumeZoneConflict: 볼륨 존 일치? │
│  ⑥ TaintToleration: Taint 통과?        │
│  ⑦ NodeAffinity: required 조건 충족?   │
│  ⑧ PodAntiAffinity: required 충돌 없음?│
│                                          │
│  → 통과한 노드만 다음 단계로            │
└─────────────────────┬───────────────────┘
                       │ 통과 노드 목록
      ┌────────────────▼────────────────────┐
      │  Phase 2: Scoring (최적 노드 선택)   │
      │                                      │
      │  각 플러그인이 0~100 점수 부여:       │
      │  ① LeastAllocated: 리소스 여유 많은  │
      │     노드일수록 고점                  │
      │  ② BalancedAllocation: CPU/메모리   │
      │     비율 균형 맞는 노드 고점         │
      │  ③ ImageLocality: 이미지가 이미       │
      │     있는 노드 고점                   │
      │  ④ InterPodAffinity: preferred       │
      │     Affinity 조건 만족 시 가산점     │
      │  ⑤ NodeAffinity: preferred 조건      │
      │     만족 시 가산점                   │
      │                                      │
      │  → 총점 최고 노드 선택              │
      │    동점 시 무작위 선택              │
      └─────────────────┬────────────────────┘
                         │
                         ▼
                    Bind (nodeName 업데이트)
```

### Filtering 주요 플러그인 상세

**① PodFitsResources**

노드의 할당 가능 리소스(Allocatable)에서 이미 배정된 파드들의 Request 합계를 뺀 값이 새 파드의 Request보다 크거나 같아야 통과한다.

```
노드 Allocatable CPU: 4000m
이미 배정된 파드 Request 합계: 3800m
새 파드 Request: 500m

3800 + 500 = 4300 > 4000 → 통과 실패 (Insufficient CPU)
```

중요한 점은 **Request를 기준으로 판단**하지, 실제 사용량이 아니다. Request 없이 파드를 만들면 0으로 계산되어 어디든 배치될 수 있다.

**⑥ TaintToleration**

노드에 Taint가 있으면, 해당 Taint를 Tolerate하는 파드만 배치할 수 있다.

```
노드에 Taint 추가:
  kubectl taint nodes worker-1 gpu=true:NoSchedule

효과:
  gpu=true Taint를 Tolerate하지 않는 파드 → worker-1에 배치 불가 (Filtering 탈락)
  
Toleration 예시:
  spec:
    tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  → 이 파드는 gpu=true:NoSchedule Taint가 있는 노드에도 배치 가능

Taint Effect 종류:
  NoSchedule        → 새 파드 배치 거부 (기존 파드 영향 없음)
  PreferNoSchedule  → 배치 비선호 (공간 없으면 허용)
  NoExecute         → 새 파드 거부 + 기존 파드 Eviction
```

**⑦ NodeAffinity**

레이블을 기준으로 특정 노드에만 배치하거나, 특정 노드를 선호하게 설정한다.

```yaml
affinity:
  nodeAffinity:
    # 필수 조건 (Filtering 단계에서 검사)
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: node-type
          operator: In
          values: ["gpu"]

    # 선호 조건 (Scoring 단계에서 가산점)
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100   # 0~100, 높을수록 가산점 크게
      preference:
        matchExpressions:
        - key: zone
          operator: In
          values: ["us-east-1a"]
```

**⑧ PodAntiAffinity**

파드 간 배치 관계를 정의한다. 같은 레이블의 파드가 이미 있는 노드를 피한다.

```yaml
affinity:
  podAntiAffinity:
    # 강제: 같은 hostname에 같은 앱 파드 2개 금지
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: my-app
      topologyKey: kubernetes.io/hostname  # 같은 노드 기준

    # 선호: 같은 zone에 몰리지 않도록 선호
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 50
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: my-app
        topologyKey: topology.kubernetes.io/zone
```

`topologyKey`는 분산의 단위를 결정한다. `kubernetes.io/hostname`이면 노드 단위, `topology.kubernetes.io/zone`이면 가용 영역 단위로 분산한다.

### Scoring 주요 플러그인 상세

**LeastAllocated (기본 활성)**

리소스 여유가 많은 노드일수록 높은 점수를 부여한다. 파드가 클러스터 전체에 고르게 분산되도록 유도한다.

```
score = (cpu_free / cpu_allocatable + mem_free / mem_allocatable) / 2 × 100

노드 A: CPU 여유 80%, 메모리 여유 70% → score = 75
노드 B: CPU 여유 20%, 메모리 여유 30% → score = 25
→ 노드 A 선택
```

**ImageLocality (기본 활성)**

이미지가 이미 다운로드되어 있는 노드에 가산점을 준다. 이미지 Pull 시간을 줄여 파드 시작 속도를 높인다.

---

## 💻 실전 실험

### 1. FailedScheduling 이벤트 분석

```bash
# 스케줄링 실패 유발 (과도한 리소스 요청)
kubectl run sched-fail --image=nginx \
  --requests='memory=100Gi'

kubectl describe pod sched-fail | grep -A10 "Events:"
# Warning  FailedScheduling  5s  default-scheduler
#   0/2 nodes are available:
#   1 Insufficient memory,    ← 1개 노드는 메모리 부족
#   1 node(s) had untolerated taint  ← 1개 노드는 Taint 문제
```

### 2. Taint와 Toleration 실험

```bash
# 노드에 Taint 추가
kubectl taint nodes kind-worker dedicated=special:NoSchedule

# Toleration 없는 파드 → 해당 노드에 배치 안 됨
kubectl run no-toleration --image=nginx
kubectl get pod no-toleration -o wide
# NODE: kind-worker2 (Taint 없는 다른 노드에 배치)

# Toleration 있는 파드 → Taint 있는 노드에도 배치 가능
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "special"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
EOF
kubectl get pod with-toleration -o wide
# NODE: kind-worker (Taint 있는 노드에 배치됨)

# Taint 제거
kubectl taint nodes kind-worker dedicated=special:NoSchedule-
```

### 3. PodAntiAffinity로 파드 강제 분산

```bash
# AntiAffinity 없는 경우: 같은 노드에 몰릴 수 있음
kubectl create deployment no-spread --image=nginx --replicas=3
kubectl get pod -l app=no-spread -o wide

# AntiAffinity로 강제 분산
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: force-spread
spec:
  replicas: 3
  selector:
    matchLabels:
      app: force-spread
  template:
    metadata:
      labels:
        app: force-spread
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: force-spread
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx
EOF

kubectl get pod -l app=force-spread -o wide
# 각 파드가 서로 다른 노드에 배치됨
# (노드 수보다 replicas가 많으면 일부가 Pending)
```

### 4. NodeAffinity로 특정 노드 그룹에만 배치

```bash
# 노드에 레이블 추가
kubectl label node kind-worker node-type=high-memory

# NodeAffinity로 해당 레이블 노드에만 배치
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-test
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values: ["high-memory"]
  containers:
  - name: nginx
    image: nginx
EOF

kubectl get pod node-affinity-test -o wide
# kind-worker 노드에만 배치됨
```

### 5. Scheduler 로그로 결정 과정 추적

```bash
kubectl logs -n kube-system kube-scheduler-kind-control-plane \
  | grep -E "Successfully assigned|FailedScheduling" | tail -10
```

---

## 📊 스케줄링 제어 방식 비교

| 방법 | 방향 | 강제/선호 | 주요 사용 사례 |
|-----|-----|---------|-------------|
| nodeSelector | 파드 → 노드 | 강제 | 단순 레이블 매칭 |
| NodeAffinity required | 파드 → 노드 | 강제 | 복잡한 조건 |
| NodeAffinity preferred | 파드 → 노드 | 선호 | 선호 존, 선호 노드 타입 |
| Taint NoSchedule | 노드 → 파드 배제 | 강제 | 특수 노드 격리 |
| Taint PreferNoSchedule | 노드 → 파드 배제 | 선호 | 유연한 격리 |
| Taint NoExecute | 노드 → 파드 퇴거 | 강제 | 노드 유지보수 |
| PodAntiAffinity required | 파드 ↔ 파드 | 강제 | HA 분산 필수 |
| PodAntiAffinity preferred | 파드 ↔ 파드 | 선호 | 최대한 분산 |
| PodAffinity | 파드 ↔ 파드 인접 | 강제/선호 | 레이턴시 최적화 |

---

## ⚖️ 트레이드오프

**required vs preferred**

`requiredDuringScheduling`은 조건을 만족하는 노드가 없으면 파드가 영구적으로 Pending된다. 예를 들어 노드가 2개인 클러스터에 `PodAntiAffinity required`로 replicas=3을 설정하면 1개가 Pending이다. `preferredDuringScheduling`은 조건이 불가능해도 배치가 진행되지만, HA 보장이 약해진다. 두 방식을 함께 사용하는 것이 일반적이다.

**Taint vs NodeAffinity**

Taint는 노드 관리자가 노드를 보호하는 방향이다. 새 파드가 실수로 특수 노드에 올라가는 것을 막는다. NodeAffinity는 파드 개발자가 파드의 요구사항을 표현하는 방향이다. GPU 파드가 "나는 GPU 노드에서만 실행돼야 해"를 스스로 선언한다. 특수 목적 노드는 Taint + NodeAffinity를 함께 사용해 양방향으로 보호하는 것이 일반적이다.

---

## 📌 핵심 정리

```
스케줄링 2단계:
  Filtering → 조건 미충족 노드 제거 (통과 못하면 Pending)
  Scoring   → 남은 노드 점수화 → 최고점 선택

Filtering 핵심 조건:
  리소스 Request 충족 (CPU, 메모리)
  Taint Toleration 통과
  NodeAffinity required 충족
  PodAntiAffinity required 비충돌

Scoring 핵심 플러그인:
  LeastAllocated → 리소스 여유 많은 노드 선호
  ImageLocality  → 이미지 캐시된 노드 가산점

HA를 위한 필수 설정:
  PodAntiAffinity required + topologyKey: kubernetes.io/hostname
  → 같은 노드에 동일 앱 파드 2개 금지
  → 노드 장애 시 서비스 유지
```

---

## 🤔 생각해볼 문제

**Q1.** `requiredDuringSchedulingIgnoredDuringExecution`의 `IgnoredDuringExecution` 부분은 무슨 의미인가?

<details>
<summary>해설 보기</summary>

스케줄링 시점에는 조건을 강제하지만, 이미 실행 중인 파드에는 적용하지 않는다는 뜻이다. 예를 들어 노드 레이블이 변경되어 NodeAffinity 조건을 더 이상 만족하지 않아도, 이미 그 노드에서 실행 중인 파드는 Evict되지 않는다. 반대 개념인 `RequiredDuringExecution`이 있다면 조건이 깨지면 즉시 Evict될 것이다. 현재 쿠버네티스에서는 Eviction을 동반하는 `RequiredDuringExecution`은 공식 구현되지 않았다.

</details>

**Q2.** 클러스터에 노드가 10개 있는데, 스케줄링 성능을 위해 Scheduler가 모든 노드를 평가하지 않을 수 있는가?

<details>
<summary>해설 보기</summary>

그렇다. 쿠버네티스 1.18부터 `percentageOfNodesToScore` 설정으로 Scoring 단계에서 평가할 노드 비율을 제한할 수 있다. 기본값은 클러스터 크기에 따라 동적으로 결정되며, 대규모 클러스터(5000+ 노드)에서는 모든 노드를 평가하는 것이 너무 느리기 때문이다. Filtering을 통과한 노드 중 일부만 무작위로 샘플링해 Scoring한다. 소규모 클러스터에서는 대부분 모든 노드를 평가한다.

</details>

**Q3.** 노드가 1개만 남은 상황에서 `PodAntiAffinity required`로 이미 배치된 파드가 있다. 새 파드를 같은 노드에 배치하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Filtering 단계에서 PodAntiAffinity required 조건을 위반하므로 해당 노드가 탈락한다. 다른 노드가 없으므로 파드는 `Pending` 상태가 된다. `FailedScheduling` 이벤트에 `1 node(s) didn't match pod anti-affinity rules`가 기록된다. 이것이 `required`와 `preferred`의 차이다. `preferred`였다면 AntiAffinity를 위반하더라도 유일한 노드에 배치된다.

</details>

---

<div align="center">

**[⬅️ 이전: 파드 생성 전 과정 — kubectl부터 컨테이너 시작까지](./01-pod-creation-full-flow.md)** | **[홈으로 🏠](../README.md)** | **[다음: 컨테이너 런타임 — containerd와 OCI 스펙 ➡️](./03-container-runtime.md)**

</div>
