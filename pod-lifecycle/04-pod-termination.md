# 파드 종료 — SIGTERM과 graceful shutdown

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `kubectl delete pod`를 실행하면 컨테이너에 어떤 시그널이 어떤 순서로 전달되는가?
- `terminationGracePeriodSeconds`는 정확히 어떤 시간을 의미하는가?
- preStop Hook은 SIGTERM 전에 실행되는가, 후에 실행되는가?
- 파드 종료 시 in-flight 요청이 끊기는 근본 원인은 무엇이고, 어떻게 방지하는가?
- SIGTERM을 무시하는 애플리케이션을 어떻게 graceful하게 종료하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

롤링 업데이트 중 일부 요청이 `502 Bad Gateway` 또는 `connection reset`으로 실패하는 현상은 파드 종료 흐름을 이해하지 못한 설정에서 비롯된다. 컨테이너가 종료되기 전에 트래픽이 차단되어야 하는데, iptables 규칙 전파 지연으로 인해 SIGTERM과 트래픽 차단이 동시에 일어나지 않는다. 이 시간차를 어떻게 처리하느냐가 무중단 배포의 핵심이다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 롤링 업데이트 중 간헐적 502 오류 발생

원리를 모를 때의 판단:
  "배포 중에는 어쩔 수 없다"
  "maxUnavailable=0으로 설정하면 해결된다"
  → maxUnavailable=0 설정 후에도 502 간헐 발생

실제 원인:
  파드 종료 시 SIGTERM과 Endpoints 제거가 동시에 발생
  iptables 규칙 전파에 1~3초 지연 발생
  → 이 시간 동안 이미 Terminating 상태인 파드로 새 요청이 유입
  → 파드는 종료 중이므로 502 응답

maxUnavailable은 Pending/Running 수를 제어할 뿐
iptables 전파 지연은 별개 문제
```

---

## ✨ 올바른 접근 (After — 종료 흐름을 이해한 설정)

```yaml
spec:
  terminationGracePeriodSeconds: 60   # 충분한 종료 시간 확보
  containers:
  - name: my-app
    lifecycle:
      preStop:
        exec:
          command: ["sleep", "5"]     # iptables 전파 대기
    # 또는 HTTP endpoint가 있는 경우:
    # preStop:
    #   httpGet:
    #     path: /shutdown
    #     port: 8080
```

```
종료 흐름 (올바른 순서):
  1. DeletionTimestamp 설정
  2. preStop Hook 실행 (sleep 5 → iptables 전파 대기)
  3. SIGTERM 전달 (애플리케이션 graceful shutdown)
  4. in-flight 요청 처리 완료 대기
  5. gracePeriod 초과 시 SIGKILL
```

---

## 🔬 내부 동작 원리

### 파드 종료 전체 흐름

```
kubectl delete pod my-pod
      │
      ▼ API Server에 DELETE 요청
1. DeletionTimestamp 설정
   Pod.metadata.deletionTimestamp = now()
   Pod.metadata.deletionGracePeriodSeconds = 30 (기본값)
   → 파드는 아직 실행 중

      │ Watch 이벤트 (MODIFIED, deletionTimestamp 설정)
      │
      ├─────────────────────────────────────────┐
      │                                         │
      ▼                                         ▼
2a. Endpoint Controller 반응              2b. kubelet 반응
    파드를 Endpoints에서 제거                preStop Hook 실행
    → kube-proxy가 iptables 갱신           (terminationGracePeriodSeconds 타이머 시작)
    (전파에 수초 소요)
      │
      ▼
3. SIGTERM 전달
   kubelet → containerd → 컨테이너 PID 1

      │
      ▼
4. 애플리케이션 graceful shutdown
   in-flight 요청 처리 완료
   새 요청 수신 거부
   DB 연결 정리

      │ terminationGracePeriodSeconds 초과 시
      ▼
5. SIGKILL
   강제 종료 (gracePeriod 무관)

      │
      ▼
6. kubelet → API Server: Pod 삭제 완료 보고
   API Server → etcd에서 Pod 오브젝트 삭제
```

### preStop Hook — SIGTERM 이전 실행

```
중요: preStop Hook은 SIGTERM보다 먼저 실행된다

실행 순서:
  DeletionTimestamp 설정
    → preStop Hook 실행 (완료까지 대기)
    → SIGTERM 전달
    → gracePeriod 종료 후 SIGKILL

타이머 주의:
  terminationGracePeriodSeconds는 preStop Hook 시작부터 카운트
  preStop에서 sleep 10을 하면:
    gracePeriod 30초 - preStop 10초 = SIGTERM 후 20초만 남음
  
  preStop 시간이 길 경우 gracePeriod 증가 필요:
  terminationGracePeriodSeconds: 60
  preStop: sleep 10  → SIGTERM 후 50초 확보
```

**preStop Hook의 목적: iptables 전파 지연 극복**

```
문제 상황:
  t=0: DeletionTimestamp 설정, SIGTERM 전달, Endpoints 제거 요청
  t=0: iptables 규칙 전파 시작 (kube-proxy에 Watch 이벤트 도달)
  t=0~3: 기존 iptables 규칙으로 여전히 Terminating 파드에 트래픽 유입
  t=3: iptables 규칙 업데이트 완료
  
  → t=0~3 사이에 유입된 요청은 이미 종료 중인 파드로 전달 → 502

preStop Hook으로 해결:
  t=0: DeletionTimestamp 설정, preStop 실행 (sleep 5)
  t=0~5: 파드는 아직 정상 운영 중 (SIGTERM 안 받음)
  t=3: iptables 규칙 전파 완료 (새 요청은 더 이상 이 파드로 안 옴)
  t=5: preStop 완료, SIGTERM 전달
  t=5~35: 애플리케이션 graceful shutdown (in-flight 요청 처리)
  t=35: 정상 종료 (또는 t=30+30=60에서 SIGKILL)
```

### SIGTERM을 무시하는 애플리케이션 처리

일부 애플리케이션(특히 shell 스크립트로 감싼 경우)은 SIGTERM을 받지 못한다.

```bash
# 문제: shell이 PID 1이 되어 SIGTERM을 자식 프로세스에 전달 안 함
CMD ["sh", "-c", "java -jar app.jar"]

# shell(PID 1)이 SIGTERM 수신 → 자식(java)에게 전달 안 함
# → gracePeriod 만료 후 SIGKILL

# 해결 1: exec 형식으로 직접 실행 (shell 없이)
CMD ["java", "-jar", "app.jar"]
# java 프로세스가 직접 PID 1 → SIGTERM 직접 수신

# 해결 2: tini 등 init 프로세스 사용
# ENTRYPOINT ["/sbin/tini", "--"]
# CMD ["java", "-jar", "app.jar"]
# tini이 PID 1, SIGTERM을 자식에게 전달, zombie 프로세스 처리

# 해결 3: preStop Hook으로 직접 종료 처리
lifecycle:
  preStop:
    exec:
      command: ["kill", "-TERM", "1"]   # PID 1에 직접 SIGTERM
```

### `kubectl delete pod --force`의 의미

```
일반 삭제:
  DeletionTimestamp 설정 → graceful 종료 대기 → etcd 삭제

강제 삭제 (--force --grace-period=0):
  etcd에서 즉시 삭제 (gracePeriod 무시)
  kubelet에는 삭제 요청이 전달되지 않을 수 있음

위험성:
  StatefulSet 파드에 --force를 사용하면
  kubelet이 아직 컨테이너 종료 전인데 etcd에서 삭제됨
  → 같은 이름의 새 파드가 생성될 수 있음
  → 두 파드가 같은 PVC에 접근하는 상황 발생 가능 (데이터 손상)

사용 권장 시나리오:
  노드가 완전히 죽어서 kubelet이 종료를 처리할 수 없는 경우에만
```

---

## 💻 실전 실험

### 1. 파드 종료 시퀀스 관찰

```bash
# 터미널 1: 파드 상태 Watch
kubectl get pod termination-test -w

# 터미널 2: 파드 생성 후 삭제
kubectl run termination-test --image=nginx
kubectl delete pod termination-test

# 터미널 1 출력:
# NAME               READY   STATUS        RESTARTS   AGE
# termination-test   1/1     Running       0          5s
# termination-test   1/1     Terminating   0          8s   ← DeletionTimestamp 설정
# (30초 후)
# termination-test   0/1     Terminating   0          38s  ← SIGTERM으로 종료됨
# (삭제 완료)
```

### 2. preStop Hook 동작 확인

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: prestop-test
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: app
    image: nginx
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "echo 'preStop running' && sleep 5 && echo 'preStop done'"]
EOF

# 삭제 시 preStop 실행 확인
kubectl delete pod prestop-test &
kubectl logs prestop-test -f
# preStop running
# preStop done
# (5초 후 SIGTERM → nginx graceful shutdown)
```

### 3. iptables 전파 지연 시뮬레이션

```bash
# Service와 Deployment 생성
kubectl create deployment drain-test --image=nginx --replicas=2
kubectl expose deployment drain-test --port=80

# 지속적으로 요청 전송
while true; do
  curl -s -o /dev/null -w "%{http_code}\n" http://$(kubectl get svc drain-test -o jsonpath='{.spec.clusterIP}')
  sleep 0.1
done &

# 롤링 업데이트 (preStop 없는 경우 502 발생 가능)
kubectl set image deployment/drain-test nginx=nginx:1.25

# 오류 확인 후 preStop 추가
kubectl patch deployment drain-test --type=merge -p='
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "nginx",
          "lifecycle": {
            "preStop": {
              "exec": {"command": ["sleep", "5"]}
            }
          }
        }]
      }
    }
  }
}'
```

### 4. SIGTERM 처리 여부 확인

```bash
# nginx는 SIGTERM을 받으면 graceful shutdown (worker 프로세스 종료 후 master 종료)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sigterm-test
spec:
  containers:
  - name: app
    image: nginx
    command: ["nginx", "-g", "daemon off;"]
EOF

# 삭제 시 nginx 로그에서 SIGTERM 처리 확인
kubectl logs sigterm-test -f &
kubectl delete pod sigterm-test

# nginx 로그:
# 2024/01/01 00:00:30 [notice] 1#1: signal 15 (SIGTERM) received, exiting
# 2024/01/01 00:00:30 [notice] 1#1: exit
```

### 5. gracePeriod 설정과 SIGKILL 타이밍

```bash
# SIGTERM을 무시하는 컨테이너 (sleep은 SIGTERM을 직접 처리)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: grace-test
spec:
  terminationGracePeriodSeconds: 10   # 10초로 설정
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "trap 'echo SIGTERM received' TERM; sleep 3600 & wait"]
EOF

time kubectl delete pod grace-test
# SIGTERM → 10초 후 SIGKILL
# real 10.3s  ← terminationGracePeriodSeconds만큼 대기
```

---

## 📊 종료 방식별 비교

| 종료 방식 | gracePeriod 준수 | 컨테이너 정리 | 사용 시나리오 |
|---------|---------------|------------|-------------|
| `kubectl delete pod` | ✅ | ✅ | 일반 삭제 |
| `kubectl delete pod --grace-period=0` | ❌ (즉시 SIGKILL) | ✅ | 응답 없는 파드 |
| `kubectl delete pod --force` | ❌ (etcd 즉시 삭제) | ⚠️ kubelet 미보장 | 노드 완전 장애 시만 |
| DeletionTimestamp (컨트롤러) | ✅ | ✅ | 롤링 업데이트, Eviction |

---

## ⚖️ 트레이드오프

**terminationGracePeriodSeconds 값 설정**

너무 짧으면 in-flight 요청 처리 중 SIGKILL로 강제 종료된다. 너무 길면 롤링 업데이트 속도가 느려진다. 애플리케이션의 최대 요청 처리 시간 × 2를 기준으로 설정하되, preStop sleep 시간을 더해야 한다. DB 연결 정리까지 고려해 넉넉히 설정하는 것이 안전하다.

**preStop sleep의 비용**

preStop sleep은 모든 파드 종료에 추가 지연을 만든다. `sleep 5`면 클러스터에서 하루 수백 번 파드가 교체될 때마다 5초씩 추가된다. 대규모 배포에서 이 지연이 누적될 수 있으나, 무중단 배포의 안정성을 위해 감수할 만한 비용이다.

---

## 📌 핵심 정리

```
파드 종료 순서:
  DeletionTimestamp 설정
  → Endpoints 제거 시작 (iptables 전파: 1~5초 소요)
  → preStop Hook 실행 (이 시간 동안 파드 정상 운영)
  → SIGTERM 전달 (애플리케이션 graceful shutdown)
  → terminationGracePeriodSeconds 만료
  → SIGKILL

무중단 배포 필수 설정:
  preStop: sleep 5~10  (iptables 전파 대기)
  terminationGracePeriodSeconds: preStop + 앱 종료 시간 + 여유
  애플리케이션: SIGTERM 핸들러 구현

SIGTERM을 못 받는 경우:
  CMD ["sh", "-c", "..."]  → exec 형식으로 변경
  CMD ["java", "-jar", "app.jar"]  → PID 1 직접 실행

--force는 StatefulSet에 절대 사용 금지
  etcd 즉시 삭제 → 동일 PVC 이중 마운트 위험
```

---

## 🤔 생각해볼 문제

**Q1.** `terminationGracePeriodSeconds: 30`으로 설정한 파드에서 preStop Hook이 40초가 걸린다면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

gracePeriod 타이머는 preStop 시작과 함께 카운트된다. 30초 후 kubelet이 SIGKILL을 보낸다. preStop Hook이 아직 실행 중이어도 컨테이너가 강제 종료된다. preStop이 오래 걸릴 것으로 예상된다면 `terminationGracePeriodSeconds`를 preStop 시간 + 앱 종료 시간보다 크게 설정해야 한다. 이 경우 최소 40 + (앱 종료 시간)으로 설정해야 한다.

</details>

**Q2.** HPA(Horizontal Pod Autoscaler)가 파드 수를 줄일 때도 같은 종료 흐름이 실행되는가?

<details>
<summary>해설 보기</summary>

그렇다. HPA가 스케일다운을 결정하면 ReplicaSet Controller가 초과 파드의 `DeletionTimestamp`를 설정한다. 이후 흐름은 `kubectl delete pod`와 동일하다. preStop Hook, gracePeriod, SIGTERM, SIGKILL 순서도 동일하게 적용된다. 따라서 프로덕션 HPA 환경에서도 preStop과 적절한 gracePeriod 설정이 필요하다.

</details>

**Q3.** Spring Boot 애플리케이션에서 server.shutdown=graceful 설정이 있으면 별도 SIGTERM 핸들러가 필요한가?

<details>
<summary>해설 보기</summary>

Spring Boot 2.3+의 `server.shutdown=graceful` 설정은 SIGTERM 수신 시 새 요청 거부 후 처리 중인 요청을 완료할 때까지 대기한다. 별도 핸들러 구현 없이 SIGTERM graceful shutdown이 동작한다. 단, preStop Hook의 sleep은 여전히 필요하다. `server.shutdown=graceful`은 SIGTERM 이후를 처리하지만, iptables 전파 지연은 SIGTERM 이전에 발생하기 때문이다. 따라서 `spring.lifecycle.timeout-per-shutdown-phase`와 `terminationGracePeriodSeconds`를 함께 조정해야 한다.

</details>

---

<div align="center">

**[⬅️ 이전: 컨테이너 런타임 — containerd와 OCI 스펙](./03-container-runtime.md)** | **[홈으로 🏠](../README.md)** | **[다음: Init Container와 Sidecar 패턴 ➡️](./05-init-container-sidecar.md)**

</div>
