# Request와 Limit — cgroups 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Request와 Limit은 각각 어떤 목적이고, 내부에서 어떻게 다르게 처리되는가?
- Scheduler는 왜 실제 사용량이 아닌 Request 합계를 기준으로 노드를 선택하는가?
- CPU Limit은 cgroups의 어떤 파라미터로 구현되는가?
- Memory Limit 초과 시 무슨 일이 발생하는가?
- Limit 없는 파드가 노드 전체에 어떤 영향을 미치는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"노드 CPU 사용량이 70%인데 파드들이 Throttling되어 응답이 느리다" — Request/Limit을 이해하면 이 역설을 설명할 수 있다. CPU Limit 설정이 Request보다 낮거나 너무 빡빡하면, 실제 노드에 여유가 있어도 파드가 Throttling된다. 반대로 Memory Limit 없이 배포하면 메모리 누수 파드가 노드 전체 메모리를 소진해 다른 파드들이 OOM으로 죽는다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황 1: CPU가 남아도는데 응답 지연 발생

원리를 모를 때의 판단:
  "노드 CPU가 20%밖에 안 되는데 왜 느리지?"
  → 네트워크 문제, DB 문제 탐색
  → 원인 못 찾음

실제 원인:
  파드 CPU Limit: 200m
  현재 CPU 사용: 210m → Throttling 발생
  노드 전체 CPU는 여유있지만 해당 파드는 할당량 초과
  → Limit을 올리거나 제거해야 함

상황 2: 밤사이 노드 전체 OOM

원리를 모를 때의 판단:
  "갑자기 트래픽이 폭증했나?"
  → 로그 확인 → 트래픽은 정상

실제 원인:
  Memory Limit 미설정 파드가 메모리 누수로 수 GB 소비
  → OOM Killer가 노드 전체 파드를 무작위 종료
  → Memory Limit 설정으로 누수 파드만 OOM 종료시켜야 함
```

---

## ✨ 올바른 접근 (After — Request와 Limit을 이해한 설정)

```
Request: "내가 최소한 이 자원을 보장받아야 한다"
  → Scheduler가 노드 배치 결정 기준
  → 이 값만큼 노드 자원이 "예약됨"
  → 실제 사용량이 Request보다 적어도 예약은 유지

Limit: "내가 이 값을 초과해서 사용하면 안 된다"
  → CPU: 초과 시 Throttling (강제 대기)
  → Memory: 초과 시 OOM Kill (컨테이너 종료)

설정 원칙:
  Request = 평균 사용량 × 1.1 (약간 여유)
  Limit = 최대 사용량 × 1.2 (버스트 허용)
  CPU Limit: 성능에 민감하면 설정 안 하거나 넉넉하게
  Memory Limit: 반드시 설정 (OOM 격리)
```

---

## 🔬 내부 동작 원리

### Request — 스케줄링 기준

```
Scheduler 노드 선택 로직:

  노드 A:
    Allocatable CPU: 4000m
    이미 배정된 파드 Request 합계: 3500m
    남은 용량: 500m

  새 파드 Request: 600m
  → 500m < 600m → 노드 A 통과 실패 (Insufficient CPU)

중요:
  Scheduler는 실제 CPU 사용량(htop에 보이는 값)을 보지 않는다
  Request 합계만 본다
  
  → 노드 CPU 실제 사용: 10% 이지만
     Request 합계가 100%이면 → 새 파드 스케줄링 불가 (Pending)

  → 모든 파드에 Request를 설정해야 Scheduler가 올바르게 동작
```

### CPU Limit — cgroups CFS 쿼터

CPU Limit은 cgroups v1의 CFS(Completely Fair Scheduler) 쿼터로 구현된다.

```
파드 CPU Limit 설정 → OCI config.json → runc → cgroups

cgroups 파라미터:
  cpu.cfs_period_us = 100000  (100ms, 고정)
  cpu.cfs_quota_us  = ?

변환 공식:
  200m CPU = 0.2 CPU = 0.2 × 100000 = 20000 us
  
  500m CPU → cpu.cfs_quota_us = 50000
  1000m (1 CPU) → cpu.cfs_quota_us = 100000
  2000m (2 CPU) → cpu.cfs_quota_us = 200000

동작:
  100ms 주기마다 컨테이너가 20ms(20000us)만 실행 가능
  20ms 초과 시 → Throttling: 나머지 80ms 동안 CPU 사용 차단
  이 기간 동안 처리 지연 발생

확인 방법:
  cat /sys/fs/cgroup/cpu/kubepods/<pod>/cpu.cfs_quota_us
  cat /sys/fs/cgroup/cpu/kubepods/<pod>/cpu.cfs_period_us

CPU Throttling 메트릭:
  container_cpu_cfs_throttled_seconds_total
  container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total
  → 30% 이상이면 CPU Limit 상향 검토
```

### CPU Request — cpu.shares

Request는 경쟁 상황에서의 상대적 우선순위를 결정한다.

```
CPU 경쟁 시 비율 배분:

  파드 A: Request 200m → cpu.shares = 204
  파드 B: Request 400m → cpu.shares = 408

  노드 CPU가 100% 사용 중일 때:
    파드 A: 200 / (200+400) = 33% 할당
    파드 B: 400 / (200+400) = 67% 할당

  노드 CPU에 여유 있을 때:
    Request 이상 사용 가능 (Limit 범위 내에서)
    cpu.shares는 경쟁 시에만 적용
```

### Memory Limit — OOM Kill

```
Memory Limit 설정 → cgroups memory.limit_in_bytes

컨테이너가 Limit 초과 시:
  Linux OOM Killer 작동
  해당 컨테이너 프로세스 kill
  kubelet이 컨테이너 재시작
  
  kubectl get pod 시 RESTARTS 카운트 증가
  kubectl describe pod → 이벤트: "OOMKilled"
  
  Status:
    containerStatuses:
    - lastState:
        terminated:
          exitCode: 137        ← SIGKILL(128+9)
          reason: OOMKilled

Memory Request:
  cgroups에 직접 적용되지 않음
  Scheduler의 노드 선택 기준으로만 사용
  메모리 Request보다 적게 사용하면 그냥 적게 씀
```

### Limit 없는 파드의 위험

```
Limit 미설정 파드:
  CPU: cpu.cfs_quota_us = -1 (무제한)
    → 노드 CPU를 원하는 만큼 사용 가능
    → CPU Throttling 없음 (좋음)
    → 다른 파드 CPU 독식 가능 (위험)

  Memory: memory.limit_in_bytes = 무제한
    → 노드 메모리를 원하는 만큼 사용 가능
    → 메모리 누수 시 노드 전체 메모리 고갈
    → OOM Killer가 무작위로 프로세스 종료
    → kubelet 자체가 OOM으로 죽을 수 있음

노드 전체 영향:
  메모리 압박 → kubelet이 QoS 클래스 순서로 파드 Eviction 시작
  (QoS → 다음 문서에서 상세 설명)
  
LimitRange로 기본값 강제:
  네임스페이스에 LimitRange 설정 시
  Request/Limit 미설정 파드에 자동으로 기본값 주입
```

---

## 💻 실전 실험

### 1. cgroups 설정 직접 확인

```bash
# CPU Limit 설정한 파드 배포
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-test
spec:
  containers:
  - name: app
    image: busybox
    command: ['sleep', '3600']
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
EOF

# 파드 UID 확인
POD_UID=$(kubectl get pod resource-test -o jsonpath='{.metadata.uid}')

# cgroups 설정 확인
docker exec -it kind-worker bash
cat /sys/fs/cgroup/cpu/kubepods/burstable/pod$POD_UID/*/cpu.cfs_quota_us
# 20000  (200m = 20000 / 100000)

cat /sys/fs/cgroup/memory/kubepods/burstable/pod$POD_UID/*/memory.limit_in_bytes
# 268435456  (256Mi = 256 * 1024 * 1024)
```

### 2. CPU Throttling 시뮬레이션

```bash
# CPU를 많이 사용하는 파드 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cpu-stress
spec:
  containers:
  - name: stress
    image: progrium/stress
    args: ['--cpu', '4', '--timeout', '60s']
    resources:
      limits:
        cpu: 200m  # 200m Limit에 4 CPU 요청 → 심한 Throttling
EOF

# Throttling 메트릭 확인 (Prometheus가 있는 경우)
# 또는 직접 cgroups 통계
POD_UID=$(kubectl get pod cpu-stress -o jsonpath='{.metadata.uid}')
docker exec kind-worker bash -c "
  cat /sys/fs/cgroup/cpu/kubepods/burstable/pod$POD_UID/*/cpu.stat
"
# nr_throttled: xxx (Throttling 발생 횟수)
# throttled_time: xxx (Throttling 총 시간 ns)
```

### 3. Memory OOM 시뮬레이션

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: mem-test
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'dd if=/dev/zero bs=1M count=300 | cat > /dev/null']
    resources:
      limits:
        memory: 100Mi  # 100Mi Limit에 300Mi 할당 시도
EOF

kubectl get pod mem-test -w
# mem-test  0/1  OOMKilled  1  30s  ← OOMKilled 확인

kubectl describe pod mem-test | grep -A5 "Last State:"
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

### 4. LimitRange 기본값 설정

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: 2
      memory: 2Gi
EOF

# Request/Limit 없이 파드 생성
kubectl run no-limit --image=nginx

# 자동으로 기본값 주입됐는지 확인
kubectl get pod no-limit -o jsonpath='{.spec.containers[0].resources}'
# {"limits":{"cpu":"500m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}
```

---

## 📊 Request vs Limit 역할 비교

| 항목 | Request | Limit |
|-----|--------|-------|
| 목적 | 보장 최솟값 | 사용 상한값 |
| Scheduler | 노드 배치 기준 | 무관 |
| cgroups (CPU) | cpu.shares (경쟁 비율) | cpu.cfs_quota_us (하드 상한) |
| cgroups (Mem) | 스케줄링 참고만 | memory.limit_in_bytes |
| 초과 시 (CPU) | 해당 없음 | Throttling |
| 초과 시 (Mem) | 해당 없음 | OOMKill |
| 미설정 (CPU) | 0으로 취급 | 무제한 |
| 미설정 (Mem) | 0으로 취급 | 무제한 |

---

## ⚖️ 트레이드오프

**CPU Limit 설정의 딜레마**

CPU Limit을 설정하면 파드가 노드에 여유가 있어도 Throttling을 당할 수 있다. 반면 미설정하면 파드 하나가 노드 CPU를 독식해 다른 파드에 영향을 줄 수 있다. 실무에서는 CPU Limit을 넉넉하게 설정하거나 Limit 없이 Request만 설정하고, Namespace ResourceQuota로 총 CPU 사용량을 제한하는 방식이 권장된다.

**Burstable과 Guaranteed의 비용**

Request == Limit인 파드(Guaranteed)는 스케줄링 시 항상 해당 자원을 예약하므로, 실제 사용량이 적어도 노드의 "할당 가능" 용량에서 차감된다. 반면 Request < Limit(Burstable)은 평소에는 Request만큼만 예약하고 필요 시 Limit까지 버스트한다. 노드 밀도(파드 수)를 높이려면 Burstable이 유리하다.

---

## 📌 핵심 정리

```
Request:
  Scheduler 기준 → 실제 사용량 아님
  cpu.shares → CPU 경쟁 시 비율 보장
  Memory → 스케줄링 참고, cgroups 적용 없음
  미설정 → 0으로 취급 → 어디든 배치될 수 있음

Limit:
  CPU: cfs_quota_us → 초과 시 Throttling (프로세스 유지)
  Memory: memory.limit_in_bytes → 초과 시 OOMKill (컨테이너 종료)
  미설정 → 무제한 → 노드 자원 독식 위험

CPU 단위: 1000m = 1 CPU core
  200m → cpu.cfs_quota_us = 20000 (100ms 중 20ms 사용 가능)

운영 원칙:
  Memory Limit 반드시 설정 (OOM 격리)
  CPU Limit: 넉넉하게 또는 Request만 설정
  LimitRange로 네임스페이스 기본값 강제
```

---

## 🤔 생각해볼 문제

**Q1.** 노드에 4 CPU가 있고, 모든 파드의 Request 합계가 4 CPU이다. 하지만 실제 사용량은 1 CPU이다. 새 파드(Request: 200m)를 배치할 수 있는가?

<details>
<summary>해설 보기</summary>

배치할 수 없다. Scheduler는 실제 사용량(1 CPU)이 아닌 Request 합계(4 CPU)를 기준으로 판단한다. 노드의 Allocatable CPU(예: 3.9 CPU, 시스템 프로세스 제외)가 이미 Request로 꽉 찼으므로, 새 200m Request 파드는 Insufficient CPU로 Pending 상태가 된다. 이 상황은 Request 값이 과도하게 높게 설정되어 실제 노드 자원 활용률이 낮은 "over-request" 상태다.

</details>

**Q2.** CPU Limit을 500m으로 설정한 파드가 갑자기 CPU 100m만 사용한다. Throttling이 발생하는가?

<details>
<summary>해설 보기</summary>

아니다. Throttling은 실제 사용량이 Limit을 초과할 때만 발생한다. 100m 사용으로 100ms 주기에서 10ms만 사용하고 90ms를 쉰다. cgroups의 cfs_quota_us(50000)를 초과하지 않으므로 Throttling이 없다. Throttling이 우려된다면 CPU 사용량이 Limit에 근접하는 경우다.

</details>

**Q3.** 같은 파드의 CPU Request는 100m인데 실제로 500m를 사용 중이다. 노드에 여유 CPU가 있다면 이 상황은 정상인가?

<details>
<summary>해설 보기</summary>

CPU Limit이 없거나 Limit이 500m 이상이라면 정상이다. CPU Request는 최솟값 보장이지 상한이 아니다. 노드에 여유 CPU가 있을 때는 Request 이상으로 사용할 수 있다. cpu.shares는 경쟁 상황에서만 적용된다. 다른 파드들이 CPU를 많이 요구하는 상황이 되면 Request 비율대로 CPU가 배분된다. 현재 노드에 여유가 있으므로 500m를 문제없이 사용할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: Chapter 4 — 스토리지 성능 고려사항](../storage/05-storage-performance.md)** | **[홈으로 🏠](../README.md)** | **[다음: QoS 클래스 — OOM 발생 시 파드 종료 순서 ➡️](./02-qos-classes.md)**

</div>
