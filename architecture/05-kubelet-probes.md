# kubelet — 노드 에이전트와 Probe

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- kubelet은 어떻게 자기 노드에 배정된 파드를 알고 컨테이너를 실행하는가?
- CRI(Container Runtime Interface)는 무엇이고, kubelet과 containerd 사이에서 어떤 역할을 하는가?
- Liveness, Readiness, Startup Probe는 각각 무엇을 판단하고, 실패 시 어떤 행동을 취하는가?
- Readiness Probe 실패는 파드를 재시작하지 않는데, 그러면 어떤 효과가 있는가?
- `kubectl describe pod`의 Events로 Probe 실패를 어떻게 추적하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

배포 직후 트래픽이 일부 오류가 나다가 정상화되는 현상, 또는 파드가 Running인데 실제로 요청을 처리하지 못하는 현상의 상당수가 Readiness Probe 미설정에서 비롯된다. 컨테이너가 시작됐다고 해서 애플리케이션이 요청을 받을 준비가 됐다는 의미가 아니다.

또한 Liveness Probe 설정이 잘못되면 부하가 높은 상황에서 멀쩡한 파드가 재시작되는 문제가 발생한다. "파드가 주기적으로 재시작된다"는 현상의 원인 중 상당수가 너무 타이트한 Liveness Probe 설정이다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 파드가 Running인데 일부 요청에서 connection refused 오류 발생

원리를 모를 때의 판단:
  "파드가 Running이니 정상이다"
  → 로드밸런서 설정 확인
  → 네트워크 팀에 문의
  → 원인 못 찾고 재배포

실제 원인:
  Readiness Probe 미설정
  → 컨테이너가 시작되는 즉시 Service의 엔드포인트에 등록됨
  → 그러나 Spring Boot 애플리케이션의 기동 시간이 20초
  → 기동 중에도 트래픽이 들어와 connection refused 발생
  
올바른 접근:
  Readiness Probe 설정 → 애플리케이션 준비 완료 후에만 트래픽 수신
  startupProbe 추가 → 초기 기동 시간을 Liveness와 분리
```

---

## ✨ 올바른 접근 (After — Probe를 이해한 설정)

```yaml
# 세 가지 Probe의 역할 분담
spec:
  containers:
  - name: my-app
    # Startup Probe: 애플리케이션이 처음 기동되는 동안 Liveness를 차단
    # (이 기간 동안 Liveness 실패해도 재시작 안 함)
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30     # 30번 실패 허용 (10초 * 30 = 최대 300초 기동 허용)
      periodSeconds: 10

    # Liveness Probe: 데드락/무한루프 등 복구 불가 상태 감지 → 재시작
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0   # startupProbe 성공 후부터 시작
      periodSeconds: 10
      failureThreshold: 3      # 3번 연속 실패 시 재시작

    # Readiness Probe: 트래픽 수신 준비 여부 → 실패 시 Service 엔드포인트에서 제거
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 3      # 3번 연속 실패 시 엔드포인트 제거
      successThreshold: 1      # 1번 성공하면 다시 엔드포인트에 추가
```

---

## 🔬 내부 동작 원리

### kubelet의 핵심 역할

kubelet은 각 노드에서 실행되는 에이전트로, Control Plane과 실제 컨테이너 실행 사이의 다리 역할을 한다.

```
kubelet의 주요 책임:
  1. API Server Watch    → 자기 노드에 배정된 파드 스펙 감지
  2. 이미지 Pull         → containerd에 이미지 다운로드 요청
  3. 컨테이너 실행       → CRI를 통해 containerd에 실행 요청
  4. Probe 실행         → HTTP/TCP/exec로 컨테이너 상태 검사
  5. 상태 보고           → 파드/컨테이너 상태를 API Server에 주기적 업데이트
  6. 리소스 관리         → cgroups로 CPU/메모리 한도 적용
  7. 볼륨 마운트         → CSI 플러그인과 협력해 볼륨 연결
```

### API Server에서 파드 스펙 받는 과정

```
1. kubelet 시작 시 API Server에 Watch 등록
   GET /api/v1/pods?fieldSelector=spec.nodeName=worker-1&watch=true

2. 새 파드가 worker-1에 배정(nodeName 설정)되면 이벤트 수신
   {"type":"ADDED","object":{"spec":{"nodeName":"worker-1",...}}}

3. Pod Manager에 파드 등록
   → 로컬 파일(/var/lib/kubelet/pods/<uid>/...)에 스펙 캐시

4. 로컬 캐시 덕분에 API Server 연결이 끊겨도
   기존 파드는 계속 실행되고 Probe도 계속 수행됨
```

### CRI(Container Runtime Interface)

kubelet은 컨테이너를 직접 실행하지 않는다. CRI라는 gRPC 인터페이스를 통해 Container Runtime에게 요청한다.

```
kubelet
  │
  │ gRPC (CRI)
  ▼
containerd
  │
  │ shim
  ▼
runc (OCI)
  │
  ▼
컨테이너 프로세스
```

CRI 덕분에 kubelet 코드를 수정하지 않고 Container Runtime을 교체할 수 있다. containerd, CRI-O 모두 CRI를 구현한다.

```bash
# kubelet이 containerd와 통신하는 소켓
ls /var/run/containerd/containerd.sock

# CRI 수준에서 컨테이너 목록 확인 (containerd 직접 조회)
crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps
```

### Probe 세 종류의 역할 분리

**Liveness Probe — "이 컨테이너가 살아있는가?"**

```
판단: 컨테이너가 복구 불가능한 상태인지 (데드락, 무한루프 등)
실패 시: 컨테이너 강제 종료 → 재시작 (restartPolicy에 따라)

실패 시나리오:
  데드락으로 HTTP 응답 불가
  → Liveness Probe 3번 연속 실패
  → kubelet이 컨테이너 kill
  → 새 컨테이너 시작

주의: 너무 타이트하게 설정하면 부하 상황에서 정상 컨테이너도 재시작됨
→ timeoutSeconds, failureThreshold를 여유있게 설정
```

**Readiness Probe — "이 컨테이너가 트래픽을 받을 준비가 됐는가?"**

```
판단: 트래픽을 처리할 수 있는 상태인지 (DB 연결 완료, 캐시 로드 완료 등)
실패 시: Service 엔드포인트에서 제거 → 트래픽 차단 (재시작 없음)
성공 시: 엔드포인트에 다시 추가 → 트래픽 수신 재개

동작 흐름:
  1. 컨테이너 시작
  2. Readiness Probe 실패 → pod.status.conditions[Readiness] = False
  3. Endpoint Controller가 감지 → Endpoints 오브젝트에서 파드 IP 제거
  4. kube-proxy가 감지 → iptables 규칙에서 해당 파드 제거
  5. 새 트래픽이 해당 파드로 라우팅되지 않음
  6. Readiness Probe 성공 → 다시 엔드포인트 추가
```

**Startup Probe — "컨테이너가 아직 기동 중인가?"**

```
판단: 처음 기동할 때 충분한 시간을 주기 위함
동작: Startup Probe 성공 전까지 Liveness Probe 비활성화

Spring Boot 애플리케이션 예시:
  기동 시간: 최대 60초
  Liveness Probe: periodSeconds=10, failureThreshold=3 → 30초 후 재시작
  문제: 60초 기동에 30초만에 Liveness가 재시작시켜 기동 루프 발생

  해결:
  Startup Probe: failureThreshold=12, periodSeconds=5 → 최대 60초 허용
  → Startup 성공 후 Liveness 활성화 → 정상 기동 보장
```

### Probe 실행 방식 세 가지

```
1. httpGet: HTTP 요청, 2xx~3xx이면 성공
   livenessProbe:
     httpGet:
       path: /healthz
       port: 8080
       httpHeaders:
       - name: Custom-Header
         value: health-check

2. tcpSocket: TCP 연결 성공 여부만 확인 (HTTP 불가능한 경우)
   livenessProbe:
     tcpSocket:
       port: 5432  ← PostgreSQL 포트

3. exec: 컨테이너 내에서 명령 실행, exit code 0이면 성공
   livenessProbe:
     exec:
       command:
       - cat
       - /tmp/healthy   ← 파일이 있으면 정상
```

### Probe 실패의 연쇄 효과

```
Liveness 실패:
  kubelet → 컨테이너 kill → 재시작
  → kubectl get pod에서 RESTARTS 카운트 증가
  → RESTARTS가 반복되면 CrashLoopBackOff (백오프 대기)

Readiness 실패:
  kubelet → pod.status.conditions 업데이트
  → Endpoint Controller → Endpoints 오브젝트 갱신
  → kube-proxy → iptables 규칙 갱신
  → Service로 들어오는 트래픽이 해당 파드를 제외하고 분배
  (Ready 파드가 0개면 트래픽을 받는 파드 없음 → 503)
```

---

## 💻 실전 실험

### 1. Probe 설정 후 동작 관찰

```bash
# Readiness Probe가 있는 Deployment 생성
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: probe-test
  template:
    metadata:
      labels:
        app: probe-test
    spec:
      containers:
      - name: nginx
        image: nginx
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 5
          failureThreshold: 2
        livenessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 10
          failureThreshold: 3
EOF

# 파드가 Ready 되기까지 관찰
kubectl get pods -w
```

### 2. Readiness Probe 실패 시 엔드포인트 제거 관찰

```bash
# Service 생성
kubectl expose deployment probe-test --port=80

# 터미널 1: Endpoints 변화 Watch
kubectl get endpoints probe-test -w

# 터미널 2: 컨테이너 내에서 nginx 중단 (Readiness Probe 실패 유발)
POD=$(kubectl get pod -l app=probe-test -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- nginx -s stop

# 터미널 1 출력:
# NAME        ENDPOINTS                         AGE
# probe-test  10.244.1.5:80,10.244.2.3:80       1m
# probe-test  10.244.2.3:80                     1m  ← 하나 제거됨
```

### 3. `kubectl describe pod`로 Probe 이벤트 추적

```bash
kubectl describe pod $POD | grep -A 20 "Events:"

# 출력:
# Events:
#   Type     Reason     Age   From     Message
#   ----     ------     ----  ----     -------
#   Warning  Unhealthy  10s   kubelet  Readiness probe failed: ...
#   Warning  Unhealthy  5s    kubelet  Readiness probe failed: ...
#   Normal   Killing    3s    kubelet  Container probe-test failed liveness probe, will be restarted
```

### 4. Liveness Probe 강제 실패로 재시작 관찰

```bash
# 건강하지 않은 상태 시뮬레이션 (파일 삭제로 exec probe 실패)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: liveness-test
spec:
  containers:
  - name: app
    image: busybox
    args: ["sh", "-c", "touch /tmp/healthy && sleep 30 && rm /tmp/healthy && sleep 600"]
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      periodSeconds: 5
      failureThreshold: 2
EOF

# 30초 후 파일이 삭제되고 Liveness 실패 → 재시작
kubectl get pod liveness-test -w
# liveness-test  1/1  Running  0  30s
# liveness-test  1/1  Running  1  42s  ← RESTARTS 증가
```

### 5. Probe 지연/타임아웃 상세 확인

```bash
# kubelet 로그에서 Probe 실행 기록
# Kind 클러스터에서는 docker exec 또는 kubectl debug 사용
kubectl get pod $POD -o jsonpath='{.status.conditions}' | python3 -m json.tool
# Ready: True/False 상태와 lastTransitionTime 확인
```

---

## 📊 Probe 유형별 비교

| 항목 | Liveness | Readiness | Startup |
|-----|---------|---------|--------|
| 실패 시 동작 | 컨테이너 재시작 | 엔드포인트 제거 | Liveness 비활성 유지 |
| 재시작 발생 | O | X | X |
| 적합한 검사 | 데드락, 무응답 | DB 연결, 초기화 완료 | 긴 기동 시간 처리 |
| 권장 failureThreshold | 3~5 | 2~3 | 기동 시간 / periodSeconds |
| initialDelaySeconds | 필요 시 | 필요 시 | 불필요 (대체 목적) |

---

## ⚖️ 트레이드오프

**Liveness Probe의 위험성**

Liveness Probe가 너무 민감하면 GC Pause, 일시적 부하 급증 같은 상황에서 멀쩡한 파드를 재시작시킨다. 재시작된 파드가 또 기동 중에 부하를 받아 Liveness 실패 → 재시작 루프가 발생할 수 있다. `timeoutSeconds`를 넉넉히 설정하고, `failureThreshold`를 3 이상으로 유지하는 것이 안전하다.

**Readiness Probe 없는 배포의 위험성**

Readiness Probe가 없으면 컨테이너가 시작되는 순간 트래픽이 들어온다. Spring Boot처럼 기동 시간이 긴 애플리케이션은 기동 중에 요청이 들어와 오류가 발생한다. 모든 프로덕션 배포에는 Readiness Probe가 필수다.

---

## 📌 핵심 정리

```
kubelet = 노드의 에이전트
  API Server Watch → 자기 노드 파드 스펙 감지
  CRI(gRPC) → containerd에 컨테이너 실행 요청
  Probe 실행 → 컨테이너 상태 지속 감시
  상태 보고 → 파드 status를 API Server에 업데이트
  API Server 끊겨도 → 기존 파드 계속 실행 (로컬 캐시)

Probe 역할 분담:
  Startup   → 기동 시간 보장 (Liveness 차단)
  Liveness  → 데드락/무응답 감지 → 재시작
  Readiness → 트래픽 수신 준비 → 엔드포인트 관리

Readiness 실패 흐름:
  kubelet → pod.status 업데이트
  → Endpoint Controller → Endpoints 갱신
  → kube-proxy → iptables 규칙 갱신
  → Service 트래픽에서 해당 파드 제외
```

---

## 🤔 생각해볼 문제

**Q1.** Readiness Probe는 통과하지만 Liveness Probe는 실패하는 상황이 가능한가? 어떤 의미인가?

<details>
<summary>해설 보기</summary>

가능하다. 예를 들어 애플리케이션이 DB 연결은 유지하고 있어 Readiness(/ready)는 200을 반환하지만, 특정 내부 로직에서 데드락이 발생해 Liveness(/healthz)는 응답하지 못하는 경우다. 이 경우 Readiness는 통과이므로 엔드포인트에 남아있고 트래픽을 받지만, 일부 요청은 데드락에 걸린 상태다. Liveness 실패로 컨테이너가 재시작되면 데드락이 해소된다. Readiness와 Liveness가 서로 다른 엔드포인트를 검사하는 이유다.

</details>

**Q2.** kubelet이 재시작되면 실행 중인 컨테이너에 어떤 영향이 있는가?

<details>
<summary>해설 보기</summary>

kubelet이 재시작되어도 이미 실행 중인 컨테이너는 계속 동작한다. 컨테이너는 containerd가 관리하는 별도의 프로세스이므로 kubelet 재시작의 영향을 받지 않는다. kubelet이 다시 시작되면 containerd에서 현재 실행 중인 컨테이너 목록을 조회해 상태를 재동기화한다. 단, kubelet이 재시작된 동안 Probe는 실행되지 않으므로 이 시간 동안 컨테이너 상태 감시에 공백이 생긴다.

</details>

**Q3.** `terminationGracePeriodSeconds`와 Readiness Probe는 어떻게 협력해 무중단 배포를 가능하게 하는가?

<details>
<summary>해설 보기</summary>

무중단 배포의 핵심 순서는: (1) Readiness Probe 실패 → 엔드포인트 제거 → 트래픽 차단, (2) SIGTERM 전달, (3) `terminationGracePeriodSeconds` 동안 in-flight 요청 처리, (4) 컨테이너 종료. 문제는 (1)과 (2)가 동시에 발생한다는 점이다. iptables 규칙 전파에 수초가 걸리는 동안에도 트래픽이 들어올 수 있다. 이를 방지하려면 `preStop` Hook에 `sleep 5` 같은 대기를 추가해 SIGTERM 전달을 지연시켜야 한다. 상세한 내용은 [Ch6-02. 무중단 배포](../deployment-operations/02-zero-downtime-deploy.md)에서 다룬다.

</details>

---

> ⬅️ 이전: [04. Controller Manager — Reconciliation Loop의 실체](./04-controller-manager-reconciliation.md)  
> ➡️ 다음: [06. 클러스터 부트스트랩 — kubeadm과 Static Pod](./06-cluster-bootstrap.md)
