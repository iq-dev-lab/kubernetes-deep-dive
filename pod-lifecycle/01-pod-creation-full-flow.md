# 파드 생성 전 과정 — kubectl부터 컨테이너 시작까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `kubectl apply`를 실행한 순간부터 컨테이너가 Running이 될 때까지 어떤 컴포넌트가 어떤 순서로 관여하는가?
- etcd에 파드 오브젝트가 저장될 때 `spec.nodeName`이 비어있는 이유는 무엇인가?
- Scheduler는 파드를 어떻게 발견하고, 어떤 방식으로 노드 배정 결과를 기록하는가?
- kubelet은 자기 노드에 배정된 파드를 어떻게 인식하고 컨테이너를 실행하는가?
- `kubectl get events`로 파드 생성 타임라인을 어떻게 재구성하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

파드가 Pending 상태에서 멈춰있을 때, "어느 단계에서 막혔는가"를 모르면 진단이 불가능하다. API Server까지 도달했는가, etcd에 저장됐는가, Scheduler가 노드를 배정했는가, kubelet이 컨테이너 실행을 시도했는가 — 각 단계를 알아야 문제 지점을 특정할 수 있다.

또한 이 흐름은 쿠버네티스의 "선언형 + 이벤트 기반" 설계 철학이 가장 잘 드러나는 지점이다. 컴포넌트들이 서로 직접 호출하지 않고, 모두 etcd 상태 변화를 Watch해 반응하는 방식이다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 파드가 Pending 상태로 수분째 멈춤

원리를 모를 때의 판단:
  "kubectl apply가 잘못됐나?" → 재적용
  "이미지 문제인가?" → 이미지 태그 확인
  → 원인 못 찾음

실제 원인별 발생 단계:
  Pending (nodeName 없음)   → Scheduler 단계 문제
    └ 원인: 리소스 부족, Taint, NodeAffinity 불일치
  Pending (nodeName 있음)   → kubelet 단계 문제
    └ 원인: 이미지 Pull 실패, PVC 마운트 실패
  ContainerCreating 지속    → 컨테이너 런타임 문제
    └ 원인: CNI 플러그인 오류, 이미지 다운로드 중

진단 명령어:
  kubectl describe pod <pod> | grep -A10 "Events:"
  kubectl get pod <pod> -o jsonpath='{.spec.nodeName}'
```

---

## ✨ 올바른 접근 (After — 단계별 진단)

```
파드 생성 5단계와 각 단계 진단 방법:

단계 1. API Server 수신 여부
  kubectl get pod → 파드가 보이면 API Server + etcd 정상

단계 2. Scheduler 배정 여부
  kubectl get pod -o wide → NODE가 비어있으면 Scheduler 미처리
  kubectl describe pod → Events에서 "FailedScheduling" 확인

단계 3. kubelet 실행 시작 여부
  kubectl describe pod → Events에서 "Pulling image" 확인
  → 없으면 kubelet이 파드를 아직 감지 못한 것

단계 4. 이미지 Pull / 컨테이너 생성
  kubectl describe pod → Events에서 ErrImagePull, CNI 오류 확인

단계 5. 컨테이너 Running
  kubectl get pod → STATUS = Running
  → 안 되면 startupProbe/livenessProbe 설정 확인
```

---

## 🔬 내부 동작 원리

### 전체 흐름 다이어그램

```
kubectl apply -f pod.yaml
      │
      ▼ HTTPS POST /api/v1/namespaces/default/pods
┌─────────────────────────────────────────────────┐
│  1. API Server                                  │
│     인증 → 인가(RBAC) → Admission Controller   │
│     → etcd에 Pod 오브젝트 저장                  │
│       spec.nodeName = ""  (미배정 상태)          │
└──────────────────────┬──────────────────────────┘
                       │ Watch 이벤트 (ADDED)
      ┌────────────────▼────────────────┐
      │  2. Scheduler                   │
      │     nodeName=""인 파드 감지      │
      │     Filtering → Scoring         │
      │     → API Server에 Bind 요청    │
      │       PATCH spec.nodeName       │
      │       = "worker-1"              │
      └────────────────┬────────────────┘
                       │ Watch 이벤트 (MODIFIED, nodeName 변경)
      ┌────────────────▼────────────────┐
      │  3. worker-1의 kubelet           │
      │     자기 노드(worker-1) 파드 감지 │
      │     → 이미지 Pull 요청           │
      │     → CNI 플러그인 호출 (IP 할당) │
      │     → containerd에 실행 요청     │
      └────────────────┬────────────────┘
                       │
      ┌────────────────▼────────────────┐
      │  4. containerd                  │
      │     containerd-shim 생성        │
      │     → runc으로 컨테이너 실행    │
      │     → namespace/cgroups 설정    │
      └────────────────┬────────────────┘
                       │
      ┌────────────────▼────────────────┐
      │  5. 상태 업데이트               │
      │     kubelet → API Server        │
      │       Pod status = Running      │
      │     kube-proxy → iptables 규칙  │
      │       새 파드 IP 반영           │
      └─────────────────────────────────┘
```

### 단계 1: API Server — 요청 수신과 etcd 저장

`kubectl apply`는 내부적으로 다음 순서로 API Server에 요청한다.

```
1. GET /api/v1/namespaces/default/pods/my-pod
   → 존재하면 PATCH(수정), 없으면 POST(생성)

2. POST /api/v1/namespaces/default/pods
   Body: Pod 스펙 (YAML → JSON 변환)

3. API Server 파이프라인:
   인증 → 인가 → Admission(MutatingWebhook → ValidatingWebhook)

4. etcd 저장:
   Key: /registry/pods/default/my-pod
   Value: protobuf 직렬화된 Pod 오브젝트
   spec.nodeName: ""   ← 아직 어느 노드에도 배정 안 됨
```

이 시점에서 파드는 `kubectl get pod`에 나타나지만, STATUS는 `Pending`이다.

### 단계 2: Scheduler — 노드 선택과 Bind

Scheduler는 API Server에 `nodeName=""`인 파드를 Watch하고 있다.

```
Watch 필터: ?fieldSelector=spec.nodeName=
            (nodeName이 빈 문자열인 파드만 수신)

파드 감지 → 스케줄링 사이클 시작:
  Filtering (부적합 노드 제거)
  → Scoring (남은 노드 점수화)
  → 최고점 노드 선택

Bind 요청:
  POST /api/v1/namespaces/default/pods/my-pod/binding
  Body: {"target": {"name": "worker-1"}}

  → API Server가 etcd의 Pod 오브젝트 업데이트
    spec.nodeName = "worker-1"
```

이 시점에서도 파드는 여전히 `Pending`이지만, `kubectl get pod -o wide`의 NODE 컬럼에 `worker-1`이 표시된다.

스케줄링 상세 알고리즘은 [02. 스케줄링 알고리즘](./02-scheduling-algorithm.md)에서 다룬다.

### 단계 3: kubelet — 파드 실행 시작

worker-1의 kubelet은 자기 노드 이름(`spec.nodeName=worker-1`)으로 필터링한 Watch를 등록해 놓았다.

```
Watch 필터: ?fieldSelector=spec.nodeName=worker-1

nodeName이 "worker-1"으로 업데이트된 MODIFIED 이벤트 수신
→ Pod Manager에 파드 추가
→ 실행 파이프라인 시작:

  1. 이미지 Pull 확인
     로컬에 이미지 있으면 Skip
     없으면 containerd에 Pull 요청
     → Event: "Pulling image nginx"
     → Event: "Successfully pulled image nginx"

  2. CNI 플러그인 호출 (네트워크 설정)
     /opt/cni/bin/ 의 CNI 플러그인 실행
     → 파드 IP 할당 (예: 10.244.1.5)
     → veth pair 생성 (파드 ↔ 노드 네트워크 연결)
     → Event: (로그 없음, 실패 시에만 Event)

  3. 볼륨 마운트
     ConfigMap, Secret, PVC 마운트

  4. containerd에 컨테이너 생성/실행 요청 (CRI)
```

### 단계 4: containerd — 컨테이너 실제 실행

kubelet은 CRI gRPC API로 containerd에 요청한다.

```
kubelet → containerd (CRI):
  RunPodSandbox()      ← pause 컨테이너(infra) 생성, 네트워크 네임스페이스 설정
  PullImage()          ← 이미지 레이어 다운로드 및 압축 해제
  CreateContainer()    ← OCI config.json 생성 (namespace, cgroups, mount 설정)
  StartContainer()     ← runc으로 컨테이너 프로세스 시작

runc 실행:
  Linux namespace 설정 (PID, Network, Mount, UTS, IPC)
  cgroups 설정 (CPU, 메모리 한도 적용)
  루트 파일시스템 마운트 (Union FS)
  컨테이너 프로세스 exec (CMD/ENTRYPOINT)
```

containerd 상세 내용은 [03. 컨테이너 런타임](./03-container-runtime.md)에서 다룬다.

### 단계 5: 상태 업데이트

컨테이너가 시작되면 kubelet이 API Server에 상태를 보고한다.

```
kubelet → PATCH /api/v1/namespaces/default/pods/my-pod/status
  status.phase: Running
  status.conditions[Ready]: True (Readiness Probe 통과 후)
  status.podIP: 10.244.1.5
  status.containerStatuses[0].state.running.startedAt: ...

Endpoint Controller (Controller Manager):
  Ready 파드의 IP를 Endpoints 오브젝트에 추가
  → kube-proxy가 Endpoints 변화 Watch
  → iptables 규칙에 10.244.1.5:PORT 추가
```

---

## 💻 실전 실험

### 1. 파드 생성 타임라인 추적

```bash
# 파드 생성
kubectl run timeline-test --image=nginx

# Events로 단계별 타임라인 확인
kubectl describe pod timeline-test | grep -A30 "Events:"

# 예상 출력:
# Events:
#   Type    Reason     Age   From               Message
#   ----    ------     ----  ----               -------
#   Normal  Scheduled  10s   default-scheduler  Successfully assigned default/timeline-test to worker-1
#   Normal  Pulling    9s    kubelet            Pulling image "nginx"
#   Normal  Pulled     7s    kubelet            Successfully pulled image "nginx" in 2s
#   Normal  Created    7s    kubelet            Created container timeline-test
#   Normal  Started    7s    kubelet            Started container timeline-test
```

### 2. 단계별 상태 변화 Watch

```bash
# 터미널 1: 파드 상태 실시간 Watch
kubectl get pod timeline-test -w

# 출력:
# NAME            READY   STATUS              RESTARTS   AGE
# timeline-test   0/1     Pending             0          0s   ← API Server 저장 완료
# timeline-test   0/1     Pending             0          0s   ← Scheduler 배정 완료
# timeline-test   0/1     ContainerCreating   0          1s   ← kubelet 실행 시작
# timeline-test   1/1     Running             0          3s   ← 컨테이너 실행 완료
```

### 3. nodeName 변화로 Scheduler 동작 확인

```bash
# Scheduler 배정 전후 nodeName 변화
kubectl get pod timeline-test -o jsonpath='{.spec.nodeName}'
# (배정 전: 빈 값)
# worker-1 (배정 후)

# Scheduler가 어느 노드를 선택했는지와 이유
kubectl get events --field-selector reason=Scheduled
# LAST SEEN  TYPE    REASON     OBJECT                    MESSAGE
# 10s        Normal  Scheduled  pod/timeline-test         Successfully assigned ... to worker-1
```

### 4. `--v=8`로 kubectl의 실제 API 호출 추적

```bash
# apply 시 어떤 API 호출이 발생하는지 확인
kubectl run api-trace-test --image=nginx --dry-run=client -v=8 2>&1 | \
  grep -E "^I|GET|POST|PATCH" | head -20
```

### 5. etcd에서 파드 원본 상태 확인

```bash
ETCD_POD=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')

# Scheduler 배정 전후 nodeName 값 확인
kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  get /registry/pods/default/timeline-test | strings | grep -A2 "nodeName"
```

### 6. Pending 원인 진단 실습

```bash
# 리소스 요청이 과도해 Pending 상태 유발
kubectl run pending-test --image=nginx \
  --requests='memory=100Gi,cpu=100'  # 클러스터 전체보다 큰 리소스 요청

kubectl describe pod pending-test | grep -A5 "Events:"
# Events:
#   Warning  FailedScheduling  5s  default-scheduler
#     0/2 nodes are available: 2 Insufficient memory.
#     preemption: 0/2 nodes are available: 2 No preemption victims found
```

---

## 📊 파드 생성 단계별 소요 시간 분석

| 단계 | 담당 컴포넌트 | 일반적 소요 시간 | 주요 변수 |
|-----|------------|--------------|---------|
| API Server 처리 | API Server | < 100ms | Admission Webhook 수 |
| Scheduler 배정 | Scheduler | < 100ms | 노드 수, 파드 수 |
| 이미지 Pull | kubelet + containerd | 1초 ~ 수분 | 이미지 크기, 네트워크 |
| CNI 설정 | kubelet + CNI | < 1초 | CNI 플러그인 종류 |
| 컨테이너 시작 | containerd + runc | < 1초 | 이미지 레이어 수 |
| Readiness 통과 | kubelet | 앱 기동 시간 | 애플리케이션 종류 |

---

## ⚖️ 트레이드오프

**이벤트 기반 아키텍처의 장점**

각 컴포넌트는 자기 관심 이벤트만 Watch하므로 느슨하게 결합된다. Scheduler가 죽어도 API Server와 kubelet은 독립적으로 동작한다. 새 컴포넌트를 추가할 때 기존 컴포넌트 수정 없이 Watch만 등록하면 된다.

**이벤트 기반 아키텍처의 단점**

이벤트 전파에 시간이 걸린다. `kubectl apply` 후 실제 파드가 Running이 되기까지 여러 컴포넌트를 거치는 지연이 발생한다. 긴급한 상황에서 이 지연이 체감될 수 있다. 또한 컴포넌트 중 하나가 비정상이면 전체 파이프라인이 해당 단계에서 멈춘다.

---

## 📌 핵심 정리

```
5단계 파드 생성 파이프라인:
  1. API Server   → etcd 저장 (nodeName="")
  2. Scheduler    → Filtering/Scoring → Bind (nodeName="worker-1")
  3. kubelet      → 이미지 Pull, CNI, containerd 실행 요청
  4. containerd   → runc으로 컨테이너 프로세스 시작
  5. kubelet      → API Server에 Running 상태 보고

각 단계에서 Pending 원인:
  nodeName 없음  → Scheduler 문제 (리소스/Taint/Affinity)
  nodeName 있음  → kubelet 문제 (이미지/CNI/볼륨)
  ContainerCreating 지속 → containerd/CNI 문제

핵심 설계 원칙:
  모든 컴포넌트가 API Server를 Watch
  서로 직접 호출하지 않음
  → 느슨한 결합, 독립적 장애 격리
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 파드 YAML을 100개 동시에 `kubectl apply`하면 Scheduler는 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

Scheduler는 내부적으로 스케줄링 큐를 유지한다. 100개의 파드 생성 요청이 들어오면 큐에 순서대로 쌓이고, Scheduler가 하나씩 처리한다. 각 파드에 대해 Filtering → Scoring → Bind가 순차적으로 실행된다. 단, 쿠버네티스 1.19부터 Scheduler는 여러 파드를 병렬로 처리하는 기능이 개선됐다. 중요한 점은 Bind 결과가 etcd에 기록된 후에야 해당 노드의 리소스가 "사용됨"으로 처리되어, 다음 파드 스케줄링에 반영된다.

</details>

**Q2.** 이미지가 이미 노드에 캐시되어 있으면 파드 생성이 얼마나 빨라지는가?

<details>
<summary>해설 보기</summary>

이미지 Pull 단계가 생략되므로 수십 MB ~ 수 GB 이미지를 받는 시간이 절약된다. nginx 같은 소형 이미지는 Pull에 2~5초가 걸리지만, 캐시된 경우 수백 ms로 줄어든다. 이 때문에 DaemonSet으로 이미지를 사전에 각 노드에 배포하거나, 이미지 Pull Policy를 `IfNotPresent`(기본값)로 유지하는 것이 중요하다. `Always`로 설정하면 매번 레지스트리를 확인하므로 지연이 생긴다.

</details>

**Q3.** `kubectl apply`와 `kubectl create`가 API Server에서 다르게 처리되는 부분은 무엇인가?

<details>
<summary>해설 보기</summary>

`kubectl create`는 단순히 POST 요청을 보내고, 이미 존재하면 `AlreadyExists(409)` 오류를 반환한다. `kubectl apply`는 Server-Side Apply(SSA) 또는 3-way merge patch를 사용한다. SSA에서 API Server는 각 필드를 "누가 마지막으로 설정했는가"를 `managedFields`에 기록하고 관리한다. 이를 통해 여러 도구(kubectl, Helm, Operator)가 같은 오브젝트의 서로 다른 필드를 수정할 때 충돌을 감지하고 관리할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: Chapter 1 — 클러스터 부트스트랩](../architecture/06-cluster-bootstrap.md)** | **[홈으로 🏠](../README.md)** | **[다음: 파드 스케줄링 알고리즘 — Filtering과 Scoring ➡️](./02-scheduling-algorithm.md)**

</div>
