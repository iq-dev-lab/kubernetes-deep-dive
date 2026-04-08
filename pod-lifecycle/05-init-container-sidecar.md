# Init Container와 Sidecar 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Init Container는 어떤 순서로 실행되고, 실패 시 어떻게 동작하는가?
- Init Container와 메인 컨테이너가 데이터를 공유하는 방법은?
- Sidecar 컨테이너가 메인 컨테이너와 네트워크 네임스페이스를 공유한다는 것이 실제로 의미하는 것은?
- Envoy가 iptables로 모든 트래픽을 가로채는 원리는 무엇인가?
- Kubernetes 1.29의 정식 Sidecar Container 기능은 기존 방식과 무엇이 다른가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Init Container는 파드 기동 순서를 제어하는 가장 신뢰할 수 있는 방법이다. "DB가 준비된 후에 앱을 시작해야 한다"는 요구사항을 `sleep` 루프로 처리하는 것은 불안정하다. Init Container로 구현하면 kubelet이 순서를 보장한다.

Sidecar 패턴은 Istio, Datadog Agent, Fluentd 같은 인프라 도구의 기반이다. 이 패턴을 이해하면 왜 Istio가 파드 내 모든 트래픽을 가로챌 수 있는지, 왜 Sidecar 추가 시 메인 컨테이너 코드를 수정할 필요가 없는지 설명할 수 있다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: Spring Boot 앱이 DB보다 먼저 시작되어 connection refused로 크래시

원리를 모를 때의 접근:
  # Deployment에 시작 명령어 지연 추가
  command: ["sh", "-c", "sleep 30 && java -jar app.jar"]
  
  문제점:
  - 30초가 충분한지 환경마다 다름
  - DB 재시작 시 앱이 먼저 올라와 같은 문제 재발
  - pod의 Start 시간이 30초 늘어남

올바른 접근:
  Init Container로 DB 접속 가능 여부를 실제로 확인
  → 실제로 연결될 때까지 대기 → 성공 시 메인 컨테이너 시작

  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres-svc 5432; do sleep 2; done']
```

---

## ✨ 올바른 접근 (After — Init Container와 Sidecar를 이해한 설계)

```
Init Container 활용 패턴:
  의존성 대기  → nc/wget으로 서비스 가용성 확인 후 진행
  설정 초기화  → 외부 Vault에서 시크릿을 받아 파일로 저장
  DB 마이그레이션 → 앱 실행 전 스키마 변경 적용 (flyway/liquibase)
  권한 설정    → 볼륨 디렉토리 권한 변경 (hostPath 등)

Sidecar 활용 패턴:
  프록시       → Istio Envoy: 모든 인바운드/아웃바운드 트래픽 제어
  로그 수집    → Fluentd: 메인 컨테이너 로그 파일을 읽어 전송
  메트릭 수집  → Datadog Agent: 메인 앱의 메트릭 엔드포인트 스크랩
  인증 프록시  → OAuth2 Proxy: 메인 앱 앞에서 인증 처리
```

---

## 🔬 내부 동작 원리

### Init Container 실행 메커니즘

kubelet은 Init Container를 정의된 순서대로 하나씩 실행한다. 각 Init Container가 성공(exit code 0)으로 완료되어야 다음 Init Container가 시작되고, 모든 Init Container가 완료되어야 메인 컨테이너들이 시작된다.

```
파드 시작 순서:

Init Container 1: wait-for-db
  → 실행 (DB 접속 대기)
  → 성공(exit 0) → 완료
      │
      ▼
Init Container 2: run-migrations
  → 실행 (DB 마이그레이션)
  → 성공(exit 0) → 완료
      │
      ▼
메인 컨테이너들 (병렬 시작):
  Container: app-server     ┐
  Container: cache-warmer   ├ 동시에 시작
  Sidecar:   envoy-proxy    ┘

Init Container 실패(exit != 0) 시:
  → kubelet이 restartPolicy에 따라 재시작
  → restartPolicy: Always/OnFailure → 성공할 때까지 재시도
  → 실패 상태가 계속되면 CrashLoopBackOff (백오프 증가)
  → 메인 컨테이너는 시작되지 않음
```

```bash
# Init Container 진행 상태 확인
kubectl get pod my-pod
# NAME     READY   STATUS      RESTARTS   AGE
# my-pod   0/1     Init:0/2    0          10s  ← Init Container 0/2 완료
# my-pod   0/1     Init:1/2    0          20s  ← Init Container 1/2 완료
# my-pod   1/1     Running     0          25s  ← 메인 컨테이너 시작

# Init Container 로그 확인
kubectl logs my-pod -c wait-for-db   # Init Container 이름으로 지정
```

### Init Container와 메인 컨테이너 간 데이터 공유

Init Container와 메인 컨테이너는 동일한 파드 내에서 볼륨을 공유할 수 있다.

```yaml
spec:
  initContainers:
  - name: fetch-config
    image: curlimages/curl
    command: ["sh", "-c", "curl https://vault/secret > /config/app.properties"]
    volumeMounts:
    - name: config-vol
      mountPath: /config         # Init Container가 설정 파일 작성

  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: config-vol
      mountPath: /etc/app        # 메인 컨테이너가 같은 볼륨에서 읽음

  volumes:
  - name: config-vol
    emptyDir: {}                 # 파드 수명과 동일한 임시 볼륨
```

`emptyDir` 볼륨은 파드가 노드에 배정될 때 생성되고, 파드가 종료될 때 삭제된다. Init Container와 메인 컨테이너가 같은 emptyDir 경로를 마운트하면 파일을 주고받을 수 있다.

### Sidecar 패턴 — 네트워크 네임스페이스 공유의 힘

파드 내 모든 컨테이너는 pause 컨테이너가 생성한 **동일한 Network namespace**를 공유한다. 이것이 Sidecar 패턴의 핵심이다.

```
파드 내 네트워크:
  pause 컨테이너 (PID 1, Network namespace 소유)
    ├── 메인 컨테이너 → pause의 Network ns 공유
    └── Sidecar 컨테이너 → pause의 Network ns 공유

결과:
  모든 컨테이너가 같은 IP 주소 사용 (10.244.1.5)
  같은 포트 공간 공유 (하나가 8080 점유하면 다른 컨테이너는 사용 불가)
  localhost로 서로 통신 가능
  
Fluentd Sidecar 예시:
  메인 앱 → /var/log/app.log에 로그 기록 (공유 emptyDir)
  Fluentd  → 같은 파일을 읽어 로그 집계 서버로 전송
  
  코드 수정 없이 로그 수집 가능 (파일 기반 연결)
```

### Istio Envoy Sidecar — iptables 트래픽 가로채기

Istio의 Sidecar 주입이 가장 정교한 Sidecar 패턴이다.

```
Istio Sidecar 주입 과정:
  1. Namespace에 istio-injection=enabled 레이블
  2. 파드 생성 시 Mutating Webhook 호출 (istio-sidecar-injector)
  3. 파드 스펙에 두 개의 컨테이너 자동 추가:
     - istio-init (Init Container): iptables 규칙 설정
     - istio-proxy (Envoy Sidecar): 실제 프록시

istio-init (Init Container) 동작:
  # 모든 인바운드 트래픽을 Envoy의 15006 포트로 리다이렉션
  iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-port 15006
  
  # 모든 아웃바운드 트래픽을 Envoy의 15001 포트로 리다이렉션
  iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-port 15001
  
  # Envoy 자체 트래픽은 제외 (무한루프 방지)
  iptables -t nat -A OUTPUT -m owner --uid-owner 1337 -j RETURN

트래픽 흐름 (Sidecar 적용 후):
  외부 요청 → 10.244.1.5:8080
    ↓ iptables PREROUTING
  Envoy 15006 포트 수신 (인바운드 처리)
    ↓ mTLS 복호화, 메트릭 수집, 정책 적용
  메인 앱 8080 포트 전달

  메인 앱 → 외부 서비스 요청
    ↓ iptables OUTPUT
  Envoy 15001 포트 수신 (아웃바운드 처리)
    ↓ mTLS 암호화, 로드밸런싱, 재시도 정책
  외부 서비스 전달
```

메인 앱 코드는 전혀 수정되지 않는다. iptables 규칙이 네트워크 네임스페이스 수준에서 트래픽을 가로채기 때문이다.

### Kubernetes 1.29 정식 Sidecar Container

기존 방식에는 문제가 있었다.

```
기존 방식의 문제:
  파드 종료 시:
    모든 컨테이너가 동시에 SIGTERM을 받음
    Sidecar(Fluentd)가 메인 앱보다 먼저 종료될 수 있음
    → 마지막 로그가 전송되지 않은 채 Sidecar 종료

  Job 파드:
    메인 컨테이너 완료 후 파드 종료 조건: 모든 컨테이너 종료
    Sidecar가 종료되지 않으면 Job이 영원히 완료되지 않음
    → 수동으로 Sidecar 종료 트릭 필요 (공유 파일로 신호 전달)

Kubernetes 1.29 SidecarContainers (정식):
  spec:
    initContainers:
    - name: envoy-proxy
      image: envoyproxy/envoy
      restartPolicy: Always   # ← 이 설정으로 Sidecar 컨테이너 선언
  
  동작 변경:
  1. Init Container 순서는 유지하되, restartPolicy: Always인 것은
     Sidecar로 처리 → 메인 컨테이너와 함께 실행 유지
  2. 종료 시 메인 컨테이너 종료 후 Sidecar 종료 (순서 보장)
  3. Job 파드: 메인 컨테이너 완료 시 Sidecar도 자동 종료
```

---

## 💻 실전 실험

### 1. Init Container 의존성 대기 패턴

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: wait-for-service
    image: busybox
    command: ['sh', '-c', 
      'until wget -qO- http://kubernetes.default.svc.cluster.local/healthz; 
       do echo "waiting..."; sleep 2; done']
  containers:
  - name: app
    image: nginx
EOF

kubectl get pod init-demo -w
# NAME        READY   STATUS     RESTARTS   AGE
# init-demo   0/1     Init:0/1   0          2s
# (서비스 접근 가능해지면)
# init-demo   0/1     PodInitializing  0   5s
# init-demo   1/1     Running          0   6s
```

### 2. Init Container와 메인 컨테이너 파일 공유

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: share-demo
spec:
  initContainers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "Hello from init" > /shared/message.txt']
    volumeMounts:
    - name: shared
      mountPath: /shared
  containers:
  - name: reader
    image: busybox
    command: ['sh', '-c', 'cat /shared/message.txt && sleep 3600']
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
EOF

kubectl logs share-demo -c reader
# Hello from init
```

### 3. Sidecar 패턴 — 로그 수집 시뮬레이션

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-log-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do echo "$(date) log line" >> /logs/app.log; sleep 1; done']
    volumeMounts:
    - name: logs
      mountPath: /logs
  - name: log-collector
    image: busybox
    command: ['sh', '-c', 'tail -f /logs/app.log']
    volumeMounts:
    - name: logs
      mountPath: /logs
  volumes:
  - name: logs
    emptyDir: {}
EOF

# log-collector Sidecar가 app 컨테이너의 로그를 실시간 수집
kubectl logs sidecar-log-demo -c log-collector -f
```

### 4. Sidecar 간 localhost 통신 확인

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: localhost-demo
spec:
  containers:
  - name: server
    image: nginx              # 80 포트에서 서비스
  - name: client
    image: curlimages/curl
    command: ['sh', '-c', 'sleep 5 && while true; do curl -s localhost:80 | head -1; sleep 3; done']
EOF

# client 컨테이너가 localhost로 server 컨테이너에 접근
kubectl logs localhost-demo -c client
# <!DOCTYPE html>   ← nginx 응답 (같은 Network namespace)
```

---

## 📊 Init Container vs Sidecar vs 메인 컨테이너 비교

| 항목 | Init Container | Sidecar | 메인 컨테이너 |
|-----|--------------|---------|------------|
| 실행 시점 | 메인 컨테이너 이전 | 메인과 동시 | 메인 |
| 완료 조건 | exit 0 필수 | 계속 실행 | 앱 로직 |
| 실패 시 | 재시도 (restartPolicy) | 재시작 | 재시작 |
| 볼륨 공유 | ✅ (emptyDir) | ✅ | ✅ |
| Network ns 공유 | ✅ | ✅ | ✅ |
| 주요 용도 | 의존성 대기, 초기화 | 프록시, 로그, 메트릭 | 비즈니스 로직 |

---

## ⚖️ 트레이드오프

**Sidecar의 리소스 오버헤드**

Sidecar 컨테이너는 추가 CPU와 메모리를 소비한다. Istio Envoy Sidecar는 파드당 약 50m CPU, 64Mi 메모리를 기본으로 사용한다. 클러스터에 1000개 파드가 있다면 Envoy만으로 50 CPU, 64 GB 메모리가 추가로 필요하다. eBPF 기반 Cilium은 Sidecar 없이 동일한 기능을 제공해 이 오버헤드를 없애는 방향으로 발전 중이다.

**Init Container의 디버깅 어려움**

Init Container가 실패해 CrashLoopBackOff가 되면, 메인 컨테이너가 시작되지 않으므로 `kubectl exec`을 사용할 수 없다. Init Container 로그(`kubectl logs my-pod -c init-container-name`)와 Events로 원인을 파악해야 한다.

---

## 📌 핵심 정리

```
Init Container:
  순서대로 실행, 각각 성공해야 다음 진행
  모두 완료 후 메인 컨테이너 시작
  emptyDir 볼륨으로 메인 컨테이너에 파일 전달
  용도: 의존성 대기, 설정 초기화, DB 마이그레이션

Sidecar 패턴의 핵심:
  같은 파드 = 같은 Network namespace = 같은 IP/localhost
  메인 앱 코드 수정 없이 기능 추가 가능
  트래픽 프록시(Envoy), 로그 수집(Fluentd), 메트릭(Datadog)

Istio Envoy 동작 원리:
  istio-init(Init Container) → iptables 규칙으로 모든 트래픽 가로채기
  envoy-proxy(Sidecar)      → 가로챈 트래픽 mTLS/정책 처리

Kubernetes 1.29 Sidecar:
  initContainers에 restartPolicy: Always 추가
  → 종료 순서 보장 (메인 먼저, Sidecar 나중)
  → Job 파드에서 Sidecar 자동 종료
```

---

## 🤔 생각해볼 문제

**Q1.** Init Container가 3개 있는데, 2번째 Init Container가 계속 실패한다. 1번째 Init Container는 몇 번 실행되는가?

<details>
<summary>해설 보기</summary>

1번만 실행된다. Init Container는 순서대로 실행되며, 각 Init Container가 성공한 후에야 다음으로 넘어간다. 1번째가 성공하면 2번째가 시작되고, 2번째가 실패하면 2번째만 재시도한다. 전체 Init Container 시퀀스가 다시 시작되지는 않는다. 단, 파드 자체가 재시작(노드 재시작 등)되면 Init Container 전체가 처음부터 다시 실행된다. 따라서 Init Container는 멱등성(여러 번 실행해도 같은 결과)이 있어야 한다.

</details>

**Q2.** Istio가 적용된 파드에서 `kubectl exec`으로 `curl localhost:8080`을 실행하면, 이 요청도 Envoy를 통과하는가?

<details>
<summary>해설 보기</summary>

`kubectl exec`으로 실행한 프로세스도 파드의 Network namespace에서 동작한다. 따라서 `curl localhost:8080`의 아웃바운드 트래픽이 iptables OUTPUT 체인에 걸려 Envoy로 리다이렉션된다. 단, Envoy 자신의 트래픽은 UID 1337 기반으로 iptables에서 제외된다. `kubectl exec`에서 실행한 프로세스는 UID 1337이 아니므로 Envoy를 통과한다. Envoy를 거치지 않으려면 직접 파드 IP와 포트를 지정해야 한다.

</details>

**Q3.** 파드에 Sidecar 컨테이너가 추가되면 파드의 Request/Limit은 어떻게 계산되는가?

<details>
<summary>해설 보기</summary>

파드의 총 Request/Limit은 모든 컨테이너(Init Container 제외)의 합산이다. 메인 컨테이너 `cpu: 200m` + Sidecar `cpu: 100m` = 파드 총 `cpu: 300m`이 된다. Scheduler는 이 합산값 기준으로 노드를 선택한다. Init Container는 순차 실행이므로 Init Container 중 가장 큰 Resource 요구사항이 Init Container 전체의 Resource로 계산된다. 단, Init Container와 메인 컨테이너는 동시 실행되지 않으므로, 파드 전체 Resource = max(Init Container 최대값, 메인+Sidecar 합산)으로 결정된다.

</details>

---

> ⬅️ 이전: [04. 파드 종료 — SIGTERM과 graceful shutdown](./04-pod-termination.md)  
> ➡️ 다음 챕터: [Ch3-01. 파드 네트워킹 기초](../networking/01-pod-networking-basics.md)
