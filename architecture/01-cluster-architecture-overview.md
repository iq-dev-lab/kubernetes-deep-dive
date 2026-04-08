# 클러스터 아키텍처 개요 — Control Plane vs Data Plane

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Control Plane과 Data Plane은 어떤 컴포넌트로 구성되고, 각각 무슨 책임을 지는가?
- 왜 모든 컴포넌트는 서로 직접 통신하지 않고 API Server를 경유하는가?
- Watch 메커니즘은 어떻게 동작하고, 폴링 방식과 무엇이 다른가?
- `kubectl apply`를 실행하면 어떤 컴포넌트가 어떤 순서로 반응하는가?
- Control Plane이 없으면 실행 중인 파드는 어떻게 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

쿠버네티스를 운영하다 보면 예상치 못한 장애 상황을 만난다. 파드가 Pending에서 멈추거나, 노드가 NotReady가 되거나, Deployment를 수정해도 변경이 반영되지 않는 상황이다. 이때 "어느 컴포넌트가 이 작업을 담당하는가"를 모르면 로그를 어디서 봐야 할지조차 알 수 없다.

클러스터 아키텍처를 이해하면 진단의 출발점이 명확해진다. 파드가 Pending이면 Scheduler 로그를 본다. 노드가 NotReady이면 해당 노드의 kubelet 로그를 본다. 컨트롤러가 동작하지 않으면 Controller Manager를 확인한다. 아키텍처가 곧 진단 지도다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: Deployment를 수정했는데 파드가 교체되지 않음

원리를 모를 때의 판단:
  "kubectl apply가 안 되는 건가?" → kubectl 재설치
  "API가 안 받는 건가?" → 다시 apply
  "캐시 문제인가?" → kubectl delete pod 강제 실행

실제 원인 (나중에 발견):
  Controller Manager 파드가 OOMKilled 상태로 재시작 루프에 빠진 상태
  API Server는 변경을 받았고 etcd에도 저장됐지만
  변경을 감지해서 새 ReplicaSet을 만들어야 할 Controller Manager가 죽어있었음

  kubectl get pods -n kube-system 한 번만 확인했으면 즉시 발견됐을 문제
```

---

## ✨ 올바른 접근 (After — 아키텍처를 알고 난 진단)

```
파드가 교체되지 않는다
  │
  ├─ API Server가 변경을 받았는가?
  │   kubectl get deployment -o yaml → resourceVersion이 바뀌었는가?
  │   → YES: API Server와 etcd는 정상
  │
  ├─ 새 ReplicaSet이 생성됐는가?
  │   kubectl get replicaset → 새 RS가 없다
  │   → Controller Manager가 동작하지 않는 것
  │
  └─ Controller Manager 상태 확인
      kubectl get pods -n kube-system | grep controller-manager
      kubectl logs -n kube-system kube-controller-manager-<node>
      → OOMKilled 발견 → 메모리 한도 조정
```

아키텍처를 알면 증상에서 원인으로 가는 경로가 보인다.

---

## 🔬 내부 동작 원리

### 전체 구조 개요

쿠버네티스 클러스터는 두 계층으로 나뉜다.

```
┌─────────────────────────────────────────────────────┐
│                   Control Plane                      │
│                                                      │
│  ┌─────────────┐  ┌──────────┐  ┌─────────────────┐ │
│  │  API Server │  │   etcd   │  │   Scheduler     │ │
│  │             │  │          │  │                 │ │
│  │ 모든 요청의  │  │ 클러스터  │  │ 파드를 어느     │ │
│  │ 진입점      │  │ 상태 저장 │  │ 노드에 배치할지 │ │
│  └──────┬──────┘  └──────────┘  └─────────────────┘ │
│         │                                            │
│  ┌──────┴──────────────────────────────────────────┐ │
│  │           Controller Manager                    │ │
│  │  ReplicaSet / Deployment / Node / Job 등        │ │
│  │  각 리소스의 Desired State를 유지하는 컨트롤러 집합 │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
                        │ API Server를 통해서만 통신
┌─────────────────────────────────────────────────────┐
│                    Data Plane                        │
│                                                      │
│  Node 1                  Node 2                      │
│  ┌──────────────────┐   ┌──────────────────┐        │
│  │ kubelet          │   │ kubelet          │        │
│  │ kube-proxy       │   │ kube-proxy       │        │
│  │ Container Runtime│   │ Container Runtime│        │
│  │ (containerd)     │   │ (containerd)     │        │
│  └──────────────────┘   └──────────────────┘        │
└─────────────────────────────────────────────────────┘
```

### Control Plane 컴포넌트

**API Server**

모든 클러스터 상태 변경의 유일한 진입점이다. etcd에 직접 접근하는 컴포넌트는 API Server뿐이며, 나머지 모든 컴포넌트는 API Server의 REST API를 통해서만 상태를 읽고 쓴다. 요청이 들어오면 인증 → 인가 → Admission Controller 파이프라인을 통과한 후 etcd에 저장한다. 상세한 내용은 [02. API Server](./02-api-server.md) 문서에서 다룬다.

**etcd**

클러스터의 모든 상태가 저장되는 분산 키-값 저장소다. `/registry/pods`, `/registry/deployments` 같은 경로에 protobuf 형식으로 저장된다. Raft 합의 알고리즘으로 분산 일관성을 보장한다. etcd가 손실되면 클러스터의 모든 상태 정보가 사라지므로, 프로덕션에서는 반드시 백업이 필요하다. 상세한 내용은 [03. etcd와 Raft](./03-etcd-raft.md)에서 다룬다.

**Scheduler**

새로 생성된 파드 중 `spec.nodeName`이 비어있는 것을 감지해 최적의 노드를 선택하고, API Server를 통해 `spec.nodeName`을 업데이트(Bind)한다. 노드 선택은 Filtering → Scoring 두 단계로 이루어진다. Scheduler가 죽어도 이미 실행 중인 파드에는 영향이 없다. 새 파드만 Pending 상태로 남는다.

**Controller Manager**

수십 개의 컨트롤러를 하나의 프로세스로 묶어 실행한다. 각 컨트롤러는 특정 리소스 종류를 감시하고, Desired State와 Actual State 사이의 차이를 좁히는 작업을 반복한다.

```
주요 컨트롤러 목록
  ReplicaSet Controller   → 파드 수 유지
  Deployment Controller   → ReplicaSet 생성/교체
  Node Controller         → 노드 상태 감시 및 조건 업데이트
  Job Controller          → 일회성 작업 실행 및 완료 추적
  Namespace Controller    → 네임스페이스 삭제 시 하위 리소스 정리
  ServiceAccount Controller → 기본 ServiceAccount 생성
```

### Data Plane 컴포넌트

**kubelet**

각 노드에서 실행되는 에이전트다. API Server를 Watch해 자기 노드에 배정된 파드 스펙을 받아오고, Container Runtime(containerd)에 컨테이너 실행을 지시한다. Liveness/Readiness Probe를 직접 실행하며, 컨테이너 상태를 API Server에 주기적으로 보고한다.

**kube-proxy**

각 노드에서 Service의 네트워크 규칙(iptables 또는 IPVS)을 관리한다. ClusterIP로 들어오는 트래픽을 실제 파드 IP로 DNAT하는 규칙을 유지한다.

**Container Runtime**

실제 컨테이너를 실행하는 소프트웨어다. kubelet은 CRI(Container Runtime Interface)라는 gRPC 인터페이스를 통해 Container Runtime과 통신한다. 현재 쿠버네티스의 기본 런타임은 containerd이며, CRI-O도 널리 사용된다.

### Watch 메커니즘 — 폴링 없이 이벤트를 받는 방법

쿠버네티스 컴포넌트들은 API Server에 지속적으로 변경 이벤트를 구독한다. 이를 Watch라고 부른다.

```
일반적인 폴링 방식 (비효율적):
  컴포넌트 → "파드 목록 줘" → API Server
  1초 후 → "파드 목록 줘" → API Server   ← 변경 없어도 매번 요청
  1초 후 → "파드 목록 줘" → API Server

Watch 방식 (쿠버네티스):
  컴포넌트 → "파드 변경 사항 구독할게" → API Server
  API Server → (파드 생성됨) → 이벤트 전송 → 컴포넌트
  API Server → (파드 삭제됨) → 이벤트 전송 → 컴포넌트
  → 변경이 생길 때만 이벤트를 받으므로 불필요한 요청 없음
```

HTTP/2 스트리밍으로 구현되며, `kubectl get pods -w`가 바로 이 Watch API를 사용하는 예시다. `resourceVersion`을 기준으로 어느 시점 이후의 변경만 받을지 지정할 수 있어, 컴포넌트가 재시작되어도 누락 없이 이벤트를 이어받을 수 있다.

### `kubectl apply`의 여정 — 전체 흐름 한눈에 보기

```
kubectl apply -f deployment.yaml
      │
      ▼
1. API Server 수신
   ├── 인증: 클라이언트 인증서 또는 Bearer Token 확인
   ├── 인가: RBAC — 이 사용자가 Deployment를 생성할 권한이 있는가?
   └── Admission: MutatingWebhook(기본값 주입) → ValidatingWebhook(유효성 검사)

      │
      ▼
2. etcd에 Desired State 저장
   /registry/deployments/default/my-app 에 protobuf로 저장

      │
      ▼
3. Controller Manager의 Deployment Controller가 Watch로 감지
   → 새 ReplicaSet 생성 요청 → API Server → etcd

4. ReplicaSet Controller가 Watch로 감지
   → 파드 3개 생성 요청 → API Server → etcd (nodeName = "")

      │
      ▼
5. Scheduler가 Watch로 감지 (nodeName이 비어있는 파드)
   → Filtering → Scoring → Bind(nodeName = "worker-1")
   → API Server → etcd 업데이트

      │
      ▼
6. worker-1의 kubelet이 Watch로 감지 (내 노드에 배정된 파드)
   → containerd에 컨테이너 실행 요청
   → CNI 플러그인으로 네트워크 설정
   → 컨테이너 시작

      │
      ▼
7. kubelet → API Server: Pod status = Running 업데이트
8. kube-proxy: 새 파드 IP를 Service iptables 규칙에 반영
```

### Control Plane 장애 시 Data Plane은?

중요한 설계 원칙이 있다. **Control Plane이 죽어도 이미 실행 중인 파드는 계속 동작한다.**

kubelet은 로컬에 파드 스펙을 캐시하고 있으며, API Server 연결이 끊어져도 컨테이너를 계속 실행하고 Probe도 계속 수행한다. 단, 다음 작업은 불가능해진다.

- 새 파드 생성 / 삭제
- 파드 재스케줄링 (노드 장애 시 파드 이동 불가)
- 설정 변경 반영
- HPA 스케일링

이 설계 덕분에 Control Plane 유지보수 중에도 서비스가 중단되지 않는다.

---

## 💻 실전 실험

### 1. 각 컴포넌트 상태 확인

```bash
# Control Plane 컴포넌트는 kube-system 네임스페이스의 Static Pod로 실행됨
kubectl get pods -n kube-system

# 예상 출력:
# etcd-kind-control-plane                      1/1     Running
# kube-apiserver-kind-control-plane            1/1     Running
# kube-controller-manager-kind-control-plane   1/1     Running
# kube-scheduler-kind-control-plane            1/1     Running
# kube-proxy-xxxxx                             1/1     Running  (각 노드마다)
# coredns-xxxxx                                1/1     Running
```

### 2. Watch 메커니즘 직접 관찰

```bash
# 터미널 1: 파드 변경을 Watch
kubectl get pods -w

# 터미널 2: 파드 생성
kubectl run test-pod --image=nginx

# 터미널 1 출력:
# NAME       READY   STATUS    RESTARTS   AGE
# test-pod   0/1     Pending   0          0s    ← Scheduler가 배정 전
# test-pod   0/1     ContainerCreating   0   1s  ← kubelet이 실행 시작
# test-pod   1/1     Running   0          2s    ← 실행 완료
```

### 3. API Server의 Watch API 직접 호출

```bash
# API Server의 Watch 엔드포인트 직접 호출
# ?watch=true 파라미터가 Watch 모드를 활성화
kubectl get --raw /api/v1/namespaces/default/pods?watch=true

# 출력 형식 (JSON 이벤트 스트림):
# {"type":"ADDED","object":{"kind":"Pod","metadata":{"name":"test-pod",...}}}
# {"type":"MODIFIED","object":{"kind":"Pod","metadata":{"name":"test-pod",...}}}
```

### 4. 컴포넌트별 로그로 아키텍처 흐름 추적

```bash
# Scheduler가 파드를 어느 노드에 배치했는지 확인
kubectl logs -n kube-system kube-scheduler-kind-control-plane | grep "Successfully assigned"
# 출력: Successfully assigned default/test-pod to kind-worker

# Controller Manager가 ReplicaSet을 생성하는 과정
kubectl logs -n kube-system kube-controller-manager-kind-control-plane | grep "Created pod"

# kubelet이 컨테이너를 실행하는 과정 (노드에 SSH 접속 후)
journalctl -u kubelet | grep "Started container"
```

### 5. Control Plane 컴포넌트 일시 중단 실험 (Kind 클러스터)

```bash
# Scheduler를 일시 중단해보기
kubectl -n kube-system delete pod kube-scheduler-kind-control-plane

# 새 파드 생성 시도
kubectl run pending-test --image=nginx

# Scheduler가 없으므로 Pending 상태 유지
kubectl get pod pending-test
# NAME           READY   STATUS    RESTARTS   AGE
# pending-test   0/1     Pending   0          30s

# Scheduler는 Static Pod이므로 kubelet이 자동으로 재시작함
# 재시작 후 Pending 파드가 자동으로 스케줄링됨
kubectl get pod pending-test -w
```

---

## 📊 컴포넌트별 장애 영향 비교

| 컴포넌트 | 장애 시 영향 | 이미 실행 중인 파드 | 새 파드 생성 |
|---------|------------|-----------------|------------|
| API Server | 모든 kubectl 명령 불가, 컴포넌트 간 통신 단절 | 계속 실행 | 불가 |
| etcd | API Server가 상태를 읽고 쓸 수 없음 | 계속 실행 | 불가 |
| Scheduler | 새 파드가 Pending에 머묾 | 계속 실행 | Pending |
| Controller Manager | Desired State 수렴 중단 | 계속 실행 | ReplicaSet 동작 안 함 |
| kubelet | 해당 노드의 파드 상태 업데이트 중단 | 계속 실행 (일정 시간) | 해당 노드 불가 |
| kube-proxy | Service 규칙 업데이트 중단 | 기존 규칙으로 계속 동작 | 새 파드 Service 반영 안 됨 |

---

## ⚖️ 트레이드오프

**중앙화된 API Server 방식의 장점**
- 모든 상태가 API Server + etcd를 통해 일관되게 관리됨
- 어느 컴포넌트든 API Server 로그 하나로 전체 흐름을 추적 가능
- 새로운 컴포넌트를 추가할 때 Watch API만 구현하면 기존 시스템과 연동 가능

**중앙화된 API Server 방식의 단점**
- API Server가 단일 장애 지점(SPOF)이 될 수 있음 → 프로덕션에서는 API Server를 다중화
- 모든 통신이 API Server를 경유하므로 대규모 클러스터에서 API Server가 병목될 수 있음
- etcd의 성능이 Control Plane 전체 성능을 결정 → [03. etcd와 Raft](./03-etcd-raft.md) 참고

---

## 📌 핵심 정리

```
Control Plane = 두뇌
  API Server        → 모든 상태 변경의 유일한 진입점, etcd 직접 접근 권한
  etcd              → 클러스터 상태의 단일 진실 공급원 (Source of Truth)
  Scheduler         → nodeName 없는 파드를 찾아 최적 노드에 배정
  Controller Manager → Desired State와 Actual State 사이의 간격을 지속적으로 좁힘

Data Plane = 실행 환경
  kubelet           → 각 노드의 에이전트, 컨테이너 실행 및 상태 보고
  kube-proxy        → Service iptables/IPVS 규칙 관리
  Container Runtime → 실제 컨테이너 실행 (containerd, CRI-O)

핵심 설계 원칙
  모든 컴포넌트는 API Server만 바라본다
  Watch 메커니즘으로 변경 이벤트를 구독 — 폴링 없음
  Control Plane 장애 시 Data Plane은 계속 동작
  쿠버네티스는 "명령형"이 아닌 "선언형" — 상태를 선언하면 컨트롤러가 맞춤
```

---

## 🤔 생각해볼 문제

**Q1.** Scheduler가 죽은 상태에서 노드가 1개 다운됐다. 해당 노드의 파드들은 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Node Controller(Controller Manager 내)가 노드 장애를 감지하고 해당 노드의 파드를 Terminating 처리한다. 그러나 새 파드를 다른 노드에 배치하는 것은 Scheduler의 역할이므로, Scheduler가 죽어있다면 새 파드는 Pending 상태로 남는다. Scheduler가 복구되면 그제야 Pending 파드들이 다른 노드에 배치된다.

</details>

**Q2.** 같은 클러스터에 Controller Manager가 2개 실행된다면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

두 Controller Manager가 동시에 같은 ReplicaSet을 조정하려 하면 중복 파드 생성 등의 충돌이 발생할 수 있다. 이를 막기 위해 Controller Manager는 Leader Election 메커니즘을 사용한다. 여러 인스턴스가 실행되어도 Leader만 실제로 컨트롤러를 실행하고, 나머지는 Leader가 죽을 때를 대기한다. `kubectl get lease -n kube-system`으로 현재 Leader를 확인할 수 있다.

</details>

**Q3.** `kubectl get pods`는 etcd를 직접 읽는가, API Server를 통해 읽는가?

<details>
<summary>해설 보기</summary>

API Server를 통해 읽는다. etcd에 직접 접근하는 컴포넌트는 API Server뿐이다. `kubectl`은 `~/.kube/config`에 설정된 API Server 주소로 REST 요청을 보내고, API Server가 etcd에서 데이터를 읽어 응답한다. API Server는 자체 인메모리 캐시(watch cache)를 유지해 etcd 직접 조회 횟수를 줄인다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: API Server — 모든 요청의 진입점 ➡️](./02-api-server.md)**

</div>
