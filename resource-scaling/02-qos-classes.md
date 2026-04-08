# QoS 클래스 — OOM 발생 시 파드 종료 순서

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Guaranteed, Burstable, BestEffort 분류 기준은 정확히 무엇인가?
- OOM Killer가 `oom_score_adj` 값을 기준으로 어떻게 파드를 선택하는가?
- kubelet이 Memory Pressure 상태에서 어떤 파드를 먼저 Evict하는가?
- `kubectl describe node`로 메모리 압박 상태를 어떻게 확인하는가?
- 중요한 파드를 OOM으로부터 보호하는 가장 효과적인 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

프로덕션 노드에서 메모리가 부족해질 때 어떤 파드가 먼저 종료되는지가 서비스 안정성을 결정한다. 핵심 API 서버 파드가 메모리 낭비 배치 작업보다 먼저 죽는다면 큰 장애가 된다. QoS 클래스와 OOM 점수를 이해하면 중요도에 따른 종료 순서를 설계할 수 있다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 배치 작업 파드가 메모리 누수로 노드 메모리 80% 소비
      → OOM Killer 작동
      → API 서버 파드가 종료됨 (배치 파드는 살아있음)

원리를 모를 때의 판단:
  "왜 배치 작업은 살아있고 API 서버가 죽었나?"
  → 랜덤이라고 생각

실제 원인:
  API 서버: Burstable (Request 200m/256Mi, Limit 없음)
  배치 작업: Guaranteed (Request == Limit, 명확하게 설정됨)
  
  OOM 점수:
    Guaranteed: oom_score_adj = -997 (거의 마지막으로 종료)
    Burstable:  oom_score_adj = 계산값 (중간)
    BestEffort: oom_score_adj = 1000 (가장 먼저 종료)

  API 서버가 Burstable이어서 배치 작업(Guaranteed)보다 먼저 종료됨

올바른 설정:
  API 서버: Guaranteed (Request == Limit)
  배치 작업: BestEffort 또는 낮은 Request/Limit
```

---

## ✨ 올바른 접근 (After — QoS를 이해한 설정)

```
서비스 중요도별 QoS 매핑:

핵심 서비스 (절대 죽으면 안 됨):
  QoS: Guaranteed (CPU Request == Limit, Memory Request == Limit)
  oom_score_adj: -997 (마지막으로 종료)

일반 서비스 (죽어도 빠르게 복구 가능):
  QoS: Burstable (Request < Limit, 또는 일부만 설정)
  oom_score_adj: 메모리 사용 비율에 따라 계산

배치/백그라운드 작업 (죽어도 재시도 가능):
  QoS: BestEffort (Request/Limit 미설정)
  oom_score_adj: 1000 (가장 먼저 종료)
```

---

## 🔬 내부 동작 원리

### QoS 클래스 분류 규칙

파드 스펙에서 자동으로 결정된다. 명시적으로 설정하는 것이 아니라, Request/Limit 설정에 따라 결정된다.

```
Guaranteed 조건:
  모든 컨테이너에 CPU Request == CPU Limit
  모든 컨테이너에 Memory Request == Memory Limit
  (Init Container 포함)

  예시:
    resources:
      requests:
        cpu: 500m
        memory: 256Mi
      limits:
        cpu: 500m    ← Request와 동일
        memory: 256Mi ← Request와 동일
  
  결과: oom_score_adj = -997

BestEffort 조건:
  모든 컨테이너에 CPU Request, Memory Request, Limit 모두 미설정

  예시:
    resources: {}  # 또는 아예 없음
  
  결과: oom_score_adj = 1000

Burstable 조건:
  위 두 조건 모두 해당 안 되는 경우
  (Request < Limit, 또는 CPU만 설정, 또는 일부 컨테이너만 설정)

  예시 1: Request < Limit
    requests: {cpu: 100m, memory: 128Mi}
    limits:   {cpu: 500m, memory: 512Mi}
  
  예시 2: Request만 설정 (Limit 없음)
    requests: {cpu: 200m, memory: 256Mi}
  
  결과: oom_score_adj = 1000 * (memory_request / node_memory) 등 계산
```

### OOM Killer와 oom_score_adj

Linux OOM Killer는 메모리가 부족할 때 `oom_score`가 높은 프로세스를 먼저 종료한다.

```
oom_score 계산:
  oom_score = 메모리 사용 비율(0~1000) + oom_score_adj

oom_score_adj 범위: -1000 ~ 1000
  -1000: 절대 종료 안 됨
  0:     기본값
  1000:  가장 먼저 종료 대상

쿠버네티스가 설정하는 oom_score_adj:

  Guaranteed 파드:
    oom_score_adj = -997
    → 거의 모든 프로세스보다 늦게 종료
    (kubelet 자신은 -999, 다음으로 중요)

  BestEffort 파드:
    oom_score_adj = 1000
    → 가장 먼저 종료 대상

  Burstable 파드:
    oom_score_adj = min(max(2, 1000 - 10 * MemoryRequest/MemoryLimit), 999)
    → Request와 Limit 비율에 따라 결정
    → Request가 Limit에 가까울수록 낮은 점수 (보호 강화)

확인 방법:
  cat /proc/<pid>/oom_score_adj
  → kubelet 프로세스: -999
  → Guaranteed 파드 컨테이너: -997
  → BestEffort 파드: 1000
```

### kubelet의 Eviction — OOM 전 선제 조치

Linux OOM Killer는 실제 메모리 고갈 시 무차별적으로 종료한다. kubelet은 이를 방지하기 위해 메모리 임계값에 도달하면 선제적으로 파드를 Evict한다.

```
Eviction 임계값 (기본값):
  memory.available < 100Mi  → Soft Eviction 시작 (grace period 후)
  memory.available < 50Mi   → Hard Eviction 시작 (즉시)
  nodefs.available < 10%    → 디스크 부족 Eviction

Eviction 순서:
  1. BestEffort 파드 먼저 Evict
  2. Burstable 중 Limit 대비 사용량이 높은 파드
  3. Guaranteed 파드 (마지막)

Eviction vs OOM Kill:
  kubelet Eviction:
    파드에 정상적으로 SIGTERM → graceful shutdown 기회
    Pod 상태: Evicted (kubectl get pod에서 확인)
    
  Linux OOM Kill:
    커널이 직접 SIGKILL (graceful 없음)
    컨테이너 즉시 종료
    kubelet이 재시작 처리
```

```bash
# 노드 Memory Pressure 상태 확인
kubectl describe node worker-1 | grep -A10 "Conditions:"
# Type              Status
# MemoryPressure    False   ← 정상
# DiskPressure      False
# PIDPressure       False

# Memory Pressure가 True면 Eviction 시작됨
# 파드들이 Evicted 상태로 kubectl get pod에 표시됨
```

---

## 💻 실전 실험

### 1. 파드 QoS 클래스 확인

```bash
# 각각 다른 QoS 클래스의 파드 생성
kubectl run guaranteed --image=nginx \
  --requests='cpu=100m,memory=128Mi' \
  --limits='cpu=100m,memory=128Mi'  # Request == Limit

kubectl run burstable --image=nginx \
  --requests='cpu=100m,memory=128Mi' \
  --limits='cpu=200m,memory=256Mi'  # Request < Limit

kubectl run besteffort --image=nginx  # No requests/limits

# QoS 클래스 확인
for pod in guaranteed burstable besteffort; do
  echo -n "$pod: "
  kubectl get pod $pod -o jsonpath='{.status.qosClass}'
  echo ""
done
# guaranteed: Guaranteed
# burstable: Burstable
# besteffort: BestEffort
```

### 2. oom_score_adj 직접 확인

```bash
# 각 파드의 컨테이너 PID 찾기
GUARANTEED_PID=$(kubectl exec guaranteed -- cat /proc/1/status | grep Pid | awk '{print $2}')
BURSTABLE_PID=$(kubectl exec burstable -- cat /proc/1/status | grep Pid | awk '{print $2}')

# 노드에서 oom_score_adj 확인
docker exec kind-worker bash

# Guaranteed 파드 컨테이너의 oom_score_adj
cat /proc/<GUARANTEED_PID>/oom_score_adj  # -997

# BestEffort 파드의 oom_score_adj
cat /proc/<BESTEFFORT_PID>/oom_score_adj  # 1000
```

### 3. Eviction 임계값 확인

```bash
# kubelet Eviction 설정 확인
docker exec kind-control-plane bash
cat /var/lib/kubelet/config.yaml | grep -A10 "eviction"

# 또는 kubectl로 kubelet 설정 확인
kubectl get configmap -n kube-system kubelet-config -o yaml | grep eviction
```

### 4. ResourceQuota로 네임스페이스 자원 제한

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-quota
  namespace: default
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    pods: "20"
EOF

# 쿼터 사용량 확인
kubectl describe resourcequota ns-quota
# Resource           Used    Hard
# limits.cpu         200m    8
# limits.memory      256Mi   8Gi
# pods               3       20
# requests.cpu       100m    4
# requests.memory    128Mi   4Gi
```

---

## 📊 QoS 클래스별 특성 비교

| QoS 클래스 | 조건 | oom_score_adj | Eviction 순서 | 적합한 워크로드 |
|----------|-----|--------------|-------------|-------------|
| Guaranteed | Request == Limit | -997 | 마지막 | 핵심 API, DB |
| Burstable | Request < Limit, 또는 일부 설정 | 2~999 (계산) | 중간 | 일반 서비스 |
| BestEffort | Request/Limit 없음 | 1000 | 가장 먼저 | 배치 작업, 백그라운드 |

---

## ⚖️ 트레이드오프

**Guaranteed의 자원 예약 비용**

Guaranteed 파드는 Request == Limit이므로, 실제 사용량이 낮아도 Limit만큼 노드 자원을 예약한다. 노드 밀도(파드 수)를 높이기 어렵다. 핵심 서비스에만 Guaranteed를 사용하고, 나머지는 Burstable로 유연하게 관리하는 것이 일반적이다.

**BestEffort의 위험성**

BestEffort 파드는 Request/Limit이 없어 자원을 유연하게 사용할 수 있지만, 메모리 압박 시 가장 먼저 종료된다. 재시도 가능한 배치 작업에는 적합하지만, 오래 실행되어야 하는 작업에는 부적합하다. BestEffort + Job의 조합(재시도 가능)이 안전한 선택이다.

---

## 📌 핵심 정리

```
QoS 클래스 결정:
  Guaranteed:  모든 컨테이너 CPU/Mem Request == Limit
  BestEffort:  모든 컨테이너 Request/Limit 없음
  Burstable:   나머지 모든 경우

OOM 종료 순서 (oom_score_adj 기반):
  BestEffort (1000) → Burstable (계산값) → Guaranteed (-997)

kubelet Eviction:
  OOM 전 선제 조치 (graceful shutdown 기회)
  memory.available < 100Mi → BestEffort부터 Evict
  OOM Kill과 달리 graceful shutdown 가능

중요 파드 보호:
  핵심 서비스 → Guaranteed (Request == Limit)
  배치 작업 → BestEffort (재시도 가능)
```

---

## 🤔 생각해볼 문제

**Q1.** 파드에 CPU Request/Limit만 설정하고 Memory Request/Limit는 없다. 이 파드의 QoS 클래스는?

<details>
<summary>해설 보기</summary>

Burstable이다. Guaranteed가 되려면 모든 컨테이너에 CPU와 Memory 모두 Request == Limit이어야 한다. Memory가 설정되지 않았으므로 Guaranteed 조건을 만족하지 못한다. BestEffort는 Request/Limit이 하나도 없어야 하는데 CPU가 설정되어 있으므로 BestEffort도 아니다. 따라서 Burstable이다.

</details>

**Q2.** Guaranteed 파드가 설정된 Memory Limit을 초과하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Guaranteed 파드여도 Memory Limit을 초과하면 OOM Kill된다. Guaranteed 분류는 "노드 전체 메모리 부족 시 보호"를 의미하지, Memory Limit 초과 시 보호를 의미하지 않는다. Limit은 cgroups의 memory.limit_in_bytes로 하드하게 적용되므로, 어떤 QoS 클래스든 Limit을 초과하면 OOM Kill된다. Guaranteed 파드에 OOM이 발생하지 않으려면 Limit을 충분히 높게 설정해야 한다.

</details>

**Q3.** DaemonSet으로 실행되는 모니터링 에이전트(Datadog, Fluentd)가 OOM으로 종료되지 않으려면 어떻게 설정해야 하는가?

<details>
<summary>해설 보기</summary>

Guaranteed QoS로 설정하는 것이 가장 확실하다. Request == Limit으로 설정하면 oom_score_adj=-997이 되어 거의 마지막으로 종료된다. 단, 모니터링 에이전트는 메모리 사용량이 가변적일 수 있으므로 Limit을 충분히 여유있게 설정해야 한다. 현실적으로는 Burstable로 설정하되 Request를 실제 평균 사용량의 2배 정도로, Limit을 피크 사용량의 150% 정도로 설정하는 것이 적절하다. VPA Recommender를 사용해 적정 값을 자동으로 계산하는 방법도 있다.

</details>

---

> ⬅️ 이전: [01. Request와 Limit — cgroups 구현](./01-request-limit-cgroups.md)  
> ➡️ 다음: [03. HPA — 메트릭 파이프라인과 스케일 알고리즘](./03-hpa.md)
