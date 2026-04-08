# 무중단 배포 — Probe와 Hook의 삼각편대

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Readiness Probe 실패 → 엔드포인트 제거 → SIGTERM 전달이 보장되지 않으면 어떤 문제가 생기는가?
- preStop Hook에서 `sleep`을 사용하는 이유와 적절한 시간은?
- `terminationGracePeriodSeconds`가 preStop Hook 시간을 포함한다는 것이 왜 중요한가?
- 새 파드가 준비되기 전에 트래픽을 받으면 어떤 오류가 발생하는가?
- 세 가지(Readiness Probe, preStop Hook, terminationGracePeriodSeconds)를 올바르게 조합하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"배포할 때마다 잠깐 502가 뜬다"는 문제의 원인 대부분이 이 세 가지 설정의 누락이나 순서 오해다. Probe, Hook, gracePeriod는 각각 독립적으로 이해하면 단순해 보이지만, 실제 배포 흐름에서는 서로 의존한다. 이 의존 관계를 모르면 각각을 아무리 잘 설정해도 순서가 맞지 않아 오류가 발생한다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: Readiness Probe와 preStop sleep 5초 설정했는데도 배포 중 502 발생

원리를 모를 때의 판단:
  "Probe가 있으니 괜찮을텐데..."
  "sleep도 5초나 되는데..."

실제 원인:
  Readiness Probe 실패 감지 → Endpoints 제거 요청
  이와 거의 동시에 DeletionTimestamp 설정 → preStop 실행
  
  iptables 규칙 전파에 1~3초 소요
  preStop sleep이 충분하지 않아 iptables 업데이트 완료 전에
  파드가 SIGTERM을 받고 커넥션 종료
  → 이 시간 동안 파드로 들어온 요청 → 502

올바른 순서:
  1. DeletionTimestamp 설정
  2. Endpoint 제거 + preStop 시작 (동시)
  3. preStop sleep 동안 iptables 전파 완료 (3~5초)
  4. preStop 완료 → SIGTERM 전달
  5. in-flight 요청 처리 완료
  6. 프로세스 종료
```

---

## ✨ 올바른 접근 (After — 세 요소의 역할과 순서 이해)

```
삼각편대 역할:

Readiness Probe:
  "나는 지금 트래픽을 받을 준비가 됐다 / 안 됐다"를 kubelet에 보고
  실패 → Endpoints에서 파드 IP 제거 → kube-proxy iptables 갱신
  → 새 파드: 준비 완료 전에 트래픽 안 받음
  → 구 파드: Evict 전에 Probe 실패로 먼저 트래픽 제거

preStop Hook:
  SIGTERM 전에 실행되는 정리 작업
  핵심 용도: "iptables 전파 지연을 기다리는 sleep"
  sleep 5~10초: 이 시간 동안 iptables 갱신 완료
  → SIGTERM을 받을 때는 새 요청이 더 이상 이 파드로 안 옴

terminationGracePeriodSeconds:
  preStop 시작부터 카운트
  이 시간이 지나면 SIGKILL 강제 종료
  설정: preStop 시간 + 앱 정상 종료 시간 + 여유
```

---

## 🔬 내부 동작 원리

### 완벽한 무중단 배포 타임라인

```
t=0: kubectl rollout 실행

    [구 파드 종료 흐름]               [새 파드 시작 흐름]
          │                                   │
          ▼                                   ▼
t=0: DeletionTimestamp 설정          t=0: 새 파드 스케줄링/시작
     Endpoints 제거 요청 시작              │
     preStop Hook 시작                     │
          │                                 │
          ▼                                 ▼
t=0~5: preStop sleep 실행            t=0~N: Startup Probe 통과 대기
       iptables 전파 완료 (3~5초)           (앱 초기화 시간)
       새 요청 더 이상 안 들어옴             │
          │                                 ▼
t=5: SIGTERM 전달                    t=N: Readiness Probe 통과
     in-flight 요청 처리 중                 Endpoints에 파드 IP 추가
          │                                 kube-proxy iptables 갱신
          ▼                                 새 파드로 트래픽 시작
t=5~35: 앱 graceful shutdown
        DB 커넥션 정리, 응답 완료
          │
          ▼
t=35: 프로세스 정상 종료 (or SIGKILL at terminationGracePeriod)
```

### 새 파드 시작 시 Probe 종류와 역할

```
Startup Probe:
  앱이 완전히 시작될 때까지 Liveness/Readiness 검사 억제
  느린 앱(JVM 15초 초기화 등)에서 false-positive Liveness kill 방지
  
  failureThreshold: 30
  periodSeconds: 10
  → 최대 300초(5분) 기다림

  startupProbe:
    httpGet:
      path: /health/startup
      port: 8080
    failureThreshold: 30
    periodSeconds: 10

Readiness Probe:
  "트래픽을 받을 준비가 됐는가?"
  실패 시: Endpoints에서 제거 (파드 Running이어도 트래픽 안 받음)
  
  readinessProbe:
    httpGet:
      path: /health/ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 5
    failureThreshold: 3   # 3번 실패 시 제거

Liveness Probe:
  "파드가 살아있는가? 데드락 상태인가?"
  실패 시: 컨테이너 재시작 (kubelet이 kill)
  
  livenessProbe:
    httpGet:
      path: /health/live
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
    failureThreshold: 3
```

### preStop Hook과 terminationGracePeriodSeconds 관계

```
terminationGracePeriodSeconds: 60  # 총 60초

  t=0:  DeletionTimestamp → preStop Hook 시작 → 타이머 시작
  t=0~10: preStop sleep 10초 (iptables 전파)
  t=10: SIGTERM 전달
  t=10~55: 앱 graceful shutdown (최대 50초)
  t=60: SIGKILL (gracePeriod 만료)

  주의:
    preStop에서 sleep 50초를 사용하면
    SIGTERM 후 남은 시간: 60 - 50 = 10초
    10초 안에 앱이 종료되지 않으면 SIGKILL

    preStop + 앱종료시간 < terminationGracePeriodSeconds
    → 설정 필수 확인 사항
```

### Readiness Probe 실패와 롤링 업데이트 연동

```
새 파드 Readiness Probe 실패가 지속될 때:

  Deployment 롤링 업데이트 중단:
    Deployment Controller: 새 파드가 Ready 안 됨 → 기존 파드 제거 중단
    → 롤링 업데이트 진행 정지

  progressDeadlineSeconds: 600 (기본 10분):
    10분 이상 진행이 없으면 → Deployment.status.conditions에 Failed 기록
    kubectl rollout status → "error: deployment ... exceeded its progress deadline"

  자동 롤백은 아님:
    Deployment 자체로는 자동 롤백 없음
    ArgoCD, Flux 같은 GitOps 도구에서 자동 롤백 구현 가능
```

---

## 💻 실전 실험

### 1. 무중단 배포 전체 설정

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zero-downtime
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        image: nginx
        lifecycle:
          preStop:
            exec:
              command: ["sleep", "10"]   # iptables 전파 대기
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 2
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
```

### 2. 배포 중 502 발생 여부 테스트

```bash
# 지속적 요청 생성
SERVICE_IP=$(kubectl get svc zero-downtime -o jsonpath='{.spec.clusterIP}')
kubectl run load --image=busybox --rm -it -- sh -c "
  while true; do
    CODE=\$(wget -qO- -S http://$SERVICE_IP 2>&1 | grep 'HTTP/' | awk '{print \$2}')
    echo \$(date +%T) \$CODE
    sleep 0.2
  done" &

# 배포 실행
kubectl set image deployment/zero-downtime app=nginx:1.25

# 502가 없으면 무중단 배포 성공
# 502가 있으면 preStop sleep 시간 또는 Probe 설정 조정 필요
```

### 3. Readiness Probe 없을 때 502 발생 시뮬레이션

```bash
# Readiness Probe 없는 Deployment
kubectl create deployment no-probe --image=nginx:1.24 --replicas=3
# (Probe 없음 → 컨테이너 시작과 동시에 Ready)

# 배포 중 오류 확인
kubectl set image deployment/no-probe no-probe=nginx:1.25

# 새 컨테이너가 아직 초기화 중에도 트래픽 받음 → 오류 가능
```

### 4. Startup Probe로 느린 앱 보호

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  failureThreshold: 30   # 30 × 10s = 최대 300초 대기
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

---

## 📊 Probe 종류별 비교

| Probe | 목적 | 실패 시 동작 | 언제 사용 |
|------|-----|-----------|---------|
| Startup | 초기화 완료 확인 | Liveness/Readiness 억제 | 느린 초기화 앱 |
| Readiness | 트래픽 수신 준비 확인 | Endpoints 제거 | 항상 설정 |
| Liveness | 파드 살아있음 확인 | 컨테이너 재시작 | 데드락 위험 앱 |

---

## ⚖️ 트레이드오프

**Readiness Probe 주기와 민감도**

`periodSeconds`가 짧으면 빠르게 감지하지만 API 서버 부하가 증가한다. `failureThreshold`가 낮으면 일시적 지연에도 파드가 Endpoints에서 제거될 수 있다. 일반적으로 `periodSeconds: 5~10`, `failureThreshold: 3`이 균형적이다.

**preStop sleep 시간**

너무 짧으면 iptables 전파가 완료되기 전에 SIGTERM이 전달된다. 너무 길면 배포 시간이 늘어난다. 클러스터 규모(노드 수, kube-proxy 반응 속도)에 따라 다르지만 일반적으로 5~10초가 권장된다. 노드가 많거나 네트워크가 느린 환경에서는 10~15초로 늘린다.

---

## 📌 핵심 정리

```
무중단 배포 삼각편대:
  Readiness Probe: 새 파드가 준비되기 전 트래픽 차단
                   구 파드 종료 전 Endpoints에서 먼저 제거
  preStop sleep:   iptables 전파 지연 대기 (5~10초)
                   이 시간 후 SIGTERM → 새 요청 없음
  terminationGracePeriodSeconds:
                   preStop + 앱 정상 종료 시간 + 여유

순서 (구 파드 종료):
  DeletionTimestamp → preStop 시작 (sleep 10)
  → iptables 갱신 완료 → SIGTERM → graceful shutdown
  → 정상 종료 or SIGKILL (gracePeriod 만료)

Probe 종류:
  Startup   → 초기화 시간 보호 (Liveness kill 방지)
  Readiness → 트래픽 수신 여부 제어
  Liveness  → 데드락/무한루프 파드 재시작
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Boot 앱에서 `/actuator/health/liveness`와 `/actuator/health/readiness`를 구분하는 이유는?

<details>
<summary>해설 보기</summary>

Liveness와 Readiness의 의미가 다르기 때문이다. Liveness는 "앱이 살아있는가 (재시작해야 하는가)"를 나타낸다. 데드락이나 무한 대기 상태에서 false를 반환한다. Readiness는 "지금 트래픽을 처리할 수 있는가"를 나타낸다. DB 연결이 끊겼거나 외부 의존성이 준비 안 됐을 때 false를 반환한다. Readiness가 false여도 Liveness는 true일 수 있다. 이를 같은 엔드포인트로 처리하면, DB 연결이 잠깐 끊겼을 때 파드가 재시작되는 문제가 발생한다.

</details>

**Q2.** terminationGracePeriodSeconds를 매우 크게 (예: 3600초) 설정하면 어떤 부작용이 있는가?

<details>
<summary>해설 보기</summary>

노드 드레인 시 해당 파드가 3600초(1시간) 동안 종료되지 않으면 노드 교체가 막힌다. 클러스터 오토스케일러의 스케일다운도 지연된다. 또한 앱이 SIGTERM 후에도 응답하지 않는 경우(버그) 1시간 동안 파드가 Terminating 상태로 유지된다. 적절한 값은 앱의 최대 요청 처리 시간 × 2 + preStop 시간으로 설정하는 것이 권장된다. 대부분의 서비스에서 30~120초로 충분하다.

</details>

**Q3.** Readiness Probe 없이 Liveness Probe만 설정하면 배포 시 어떤 위험이 있는가?

<details>
<summary>해설 보기</summary>

Readiness Probe 없으면 컨테이너가 시작되는 순간 자동으로 Ready로 처리된다. 롤링 업데이트 중 새 파드가 아직 애플리케이션 초기화 중(DB 연결, 캐시 워밍 등)인데도 트래픽이 들어온다. 이 요청들은 아직 준비 안 된 앱에서 오류로 처리된다. Liveness만 있으면 앱이 "살아있음"은 알 수 있지만 "서비스 가능"은 보장하지 못한다. Readiness Probe는 항상 설정해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: Deployment 롤링 업데이트 — ReplicaSet 교체 알고리즘](./01-rolling-update.md)** | **[홈으로 🏠](../README.md)** | **[다음: RBAC — 최소 권한 원칙 구현 ➡️](./03-rbac.md)**

</div>
