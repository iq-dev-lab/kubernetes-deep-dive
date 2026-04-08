# Controller Manager — Reconciliation Loop의 실체

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Reconciliation Loop는 정확히 무엇이고, 어떤 주기로 실행되는가?
- Desired State와 Actual State를 어떻게 비교하고, 차이가 생기면 무엇을 하는가?
- ReplicaSet Controller가 파드 수를 유지하는 정확한 메커니즘은?
- "선언형(Declarative)"이 내부에서 의미하는 것은 무엇인가?
- Controller Manager 내에 몇 개의 컨트롤러가 있고, 각각 무슨 역할인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

쿠버네티스가 "자가 치유(self-healing)"한다고 말할 때, 그 실체가 바로 Reconciliation Loop다. 파드가 죽어도 자동으로 재생성되고, 노드가 다운되어도 다른 노드에서 파드가 올라오는 것이 모두 이 루프의 결과다.

이 메커니즘을 이해하면 단순히 기능을 사용하는 것을 넘어, 커스텀 컨트롤러(Operator)를 작성하거나, 특정 상황에서 왜 파드가 예상대로 복구되지 않는지 진단할 수 있다. 또한 CRD(Custom Resource Definition)와 Operator 패턴의 기반 원리이기도 하다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: kubectl delete pod my-app-xxx 했는데 파드가 즉시 재생성됨
      "삭제가 안 된다!"

원리를 모를 때의 판단:
  "kubectl delete가 이상하다"
  "캐시 문제다" → kubectl delete pod --force --grace-period=0
  → 삭제됐지만 다시 생김
  "뭔가 자동 복구 스크립트가 실행되고 있나?" → cron 확인

실제 원인:
  Deployment(또는 ReplicaSet)가 파드 3개를 유지하도록 설정됨
  ReplicaSet Controller가 파드 수가 2개로 줄어든 것을 감지
  → 새 파드 생성 요청

올바른 삭제 방법:
  kubectl delete deployment my-deployment
  또는 kubectl scale deployment my-deployment --replicas=0
  (ReplicaSet Controller가 관리하는 상위 오브젝트를 조작해야 함)
```

---

## ✨ 올바른 접근 (After — Reconciliation Loop를 알고 난 운영)

```
쿠버네티스는 "현재 상태를 명령하는" 시스템이 아니라
"원하는 상태를 선언하면 컨트롤러가 맞춰주는" 시스템

명령형 (Imperative):
  "파드 3개를 지금 당장 실행해라"
  → 실행 후 시스템은 관심 없음. 죽으면 그냥 죽음

선언형 (Declarative):
  "파드가 항상 3개 실행된 상태여야 한다"
  → 컨트롤러가 계속 감시. 2개가 되면 1개 더 만듦. 4개가 되면 1개 지움

실무 적용:
  수동으로 파드를 삭제하거나 조작하지 말 것
  항상 Deployment, StatefulSet 등 상위 오브젝트를 수정
  변경은 YAML 파일로 → kubectl apply → 컨트롤러가 수렴
```

---

## 🔬 내부 동작 원리

### Reconciliation Loop의 기본 구조

모든 컨트롤러는 동일한 패턴으로 동작한다.

```
무한 루프:
  1. Desired State 읽기    (etcd에서 원하는 상태 조회)
  2. Actual State 읽기     (현재 실제 상태 조회)
  3. 차이(Diff) 계산
  4. 차이를 좁히는 작업 수행
  5. 잠시 대기 후 1로 돌아감

의사 코드:
  for {
      desired := getDesiredState()   // etcd에서 ReplicaSet 스펙 읽기
      actual  := getActualState()    // 현재 실행 중인 파드 목록 읽기
      diff    := desired - actual

      if diff > 0 { createPods(diff) }   // 부족하면 생성
      if diff < 0 { deletePods(-diff) }  // 초과하면 삭제

      time.Sleep(resyncPeriod)
  }
```

실제로는 이벤트 기반으로 동작한다. 매초 폴링하는 것이 아니라, Watch 메커니즘으로 관련 오브젝트가 변경될 때만 Reconcile 함수가 호출된다.

### ReplicaSet Controller 상세 동작

```
ReplicaSet Controller 작업 흐름:

1. Watch 등록
   └── API Server에서 Pod와 ReplicaSet 변경 이벤트 구독

2. 이벤트 수신 (파드 하나 삭제됨)
   └── 큐(Work Queue)에 해당 ReplicaSet 추가

3. Reconcile 함수 실행
   a. etcd에서 ReplicaSet 스펙 조회
      spec.replicas: 3

   b. 현재 파드 목록 조회 (레이블 셀렉터로)
      selector: app=my-app
      → 현재 파드: 2개 (1개가 삭제됨)

   c. 차이 계산: 3 - 2 = 1 (파드 1개 부족)

   d. 새 파드 스펙 생성
      spec.template을 기반으로 Pod 오브젝트 생성
      metadata.ownerReferences: [ReplicaSet my-rs] 설정

   e. API Server에 Pod 생성 요청
      POST /api/v1/namespaces/default/pods

   f. etcd에 저장 → Scheduler가 Watch로 감지 → 노드 배정
```

**ownerReferences의 역할**

파드의 `metadata.ownerReferences`에 소유 ReplicaSet이 기록된다. 이를 통해:

- ReplicaSet 삭제 시 파드도 연쇄 삭제(Garbage Collection)
- 파드가 고아(orphan)가 됐을 때 어느 컨트롤러가 관리해야 하는지 판단
- `kubectl get pod --show-labels`로 레이블 확인 후, 레이블을 제거하면 ReplicaSet 관리에서 벗어남 (디버깅에 활용)

```bash
# 파드의 ownerReferences 확인
kubectl get pod my-app-xxx -o jsonpath='{.metadata.ownerReferences}'
# [{"apiVersion":"apps/v1","kind":"ReplicaSet","name":"my-deployment-xxx","uid":"..."}]
```

### Controller Manager 내의 주요 컨트롤러

Controller Manager는 단일 프로세스 안에서 수십 개의 컨트롤러를 고루틴(goroutine)으로 실행한다.

```
kube-controller-manager 프로세스
  │
  ├── Deployment Controller
  │     - ReplicaSet 생성/업데이트/삭제
  │     - 롤링 업데이트 조율
  │
  ├── ReplicaSet Controller
  │     - 파드 수 유지
  │     - 초과/부족 파드 조정
  │
  ├── StatefulSet Controller
  │     - 순서 있는 파드 생성/삭제
  │     - 안정적인 네트워크 ID 유지
  │
  ├── DaemonSet Controller
  │     - 각 노드에 1개 파드 보장
  │     - 새 노드 추가 시 자동 배포
  │
  ├── Job Controller
  │     - 일회성 작업 완료 추적
  │     - 실패 시 재시도
  │
  ├── Node Controller
  │     - 노드 상태 감시
  │     - NotReady 노드의 파드 eviction
  │     - 노드 조건(Condition) 업데이트
  │
  ├── Endpoint Controller
  │     - Service와 파드 IP 연결 (Endpoints 오브젝트 유지)
  │
  ├── Namespace Controller
  │     - 네임스페이스 삭제 시 하위 리소스 정리
  │
  ├── ServiceAccount Controller
  │     - 새 네임스페이스에 default ServiceAccount 자동 생성
  │
  └── PersistentVolume Controller
        - PVC와 PV 바인딩
        - 동적 프로비저닝 요청
```

### Node Controller와 파드 Eviction

Node Controller는 kubelet의 상태 업데이트를 기반으로 노드 상태를 추적한다.

```
노드 장애 감지 흐름:

1. kubelet이 API Server에 주기적으로 상태 보고 (기본 10초)

2. 40초간 보고 없음 (node-monitor-grace-period)
   → Node 조건: Ready → Unknown

3. 5분 후에도 복구 없음 (pod-eviction-timeout)
   → 해당 노드의 모든 파드에 DeletionTimestamp 설정
   → 새 파드들이 다른 노드에 스케줄링됨

(파드가 즉시 이동하지 않는 이유: 네트워크 순간 단절과 실제 장애를 구분하기 위해 유예 시간 필요)
```

### Leader Election — 중복 실행 방지

Controller Manager가 다중화되어 있어도 한 번에 하나만 실제로 동작한다.

```bash
# 현재 Leader 확인
kubectl get lease -n kube-system kube-controller-manager -o yaml
# spec:
#   holderIdentity: kind-control-plane  ← 현재 Leader 노드
#   leaseDurationSeconds: 15
#   renewTime: "2024-01-01T00:00:00Z"
```

Leader는 15초마다 Lease를 갱신한다. 15초(leaseDuration) 안에 갱신이 없으면 다른 인스턴스가 Leader가 된다.

---

## 💻 실전 실험

### 1. Reconciliation Loop 직접 관찰

```bash
# 터미널 1: 파드 변화 Watch
kubectl get pods -w

# 터미널 2: Deployment 생성
kubectl create deployment test-dep --image=nginx --replicas=3

# 터미널 1 출력 (순서 관찰):
# test-dep-xxx-1   0/1  Pending  → ContainerCreating → Running
# test-dep-xxx-2   0/1  Pending  → ContainerCreating → Running
# test-dep-xxx-3   0/1  Pending  → ContainerCreating → Running

# 터미널 2: 파드 하나 강제 삭제
kubectl delete pod test-dep-xxx-1

# 터미널 1: ReplicaSet Controller가 즉시 새 파드 생성
# test-dep-xxx-1   1/1  Running  → Terminating
# test-dep-xxx-4   0/1  Pending  → ContainerCreating → Running
```

### 2. Controller Manager 로그로 Reconciliation 추적

```bash
# ReplicaSet Controller의 파드 생성 로그
kubectl logs -n kube-system kube-controller-manager-kind-control-plane \
  | grep -i "replicaset" | tail -20

# Node Controller가 노드 상태를 갱신하는 로그
kubectl logs -n kube-system kube-controller-manager-kind-control-plane \
  | grep -i "node" | grep -i "condition" | tail -10
```

### 3. ownerReferences로 소유 관계 추적

```bash
# Deployment → ReplicaSet → Pod 소유 관계 확인
kubectl get pod test-dep-xxx -o jsonpath='{.metadata.ownerReferences[0].name}'
# test-dep-<hash>  ← ReplicaSet 이름

kubectl get replicaset test-dep-<hash> -o jsonpath='{.metadata.ownerReferences[0].name}'
# test-dep  ← Deployment 이름
```

### 4. 레이블 제거로 파드를 컨트롤러 관리에서 분리

```bash
# 파드를 디버깅하기 위해 ReplicaSet 관리에서 분리하는 방법
# (ReplicaSet은 새 파드를 생성하지만, 이 파드는 계속 살아있음)
kubectl label pod test-dep-xxx-1 app-  # app 레이블 제거

# ReplicaSet Controller가 파드가 1개 줄었다고 판단하여 새 파드 생성
# 원래 파드는 레이블 없이 살아있음 → 격리된 상태로 디버깅 가능
kubectl get pods --show-labels
```

### 5. Deployment 롤링 업데이트의 ReplicaSet 교체 관찰

```bash
# 현재 ReplicaSet 확인
kubectl get replicaset

# 이미지 업데이트 → Deployment Controller가 새 ReplicaSet 생성
kubectl set image deployment/test-dep nginx=nginx:1.25

# ReplicaSet 변화 관찰
kubectl get replicaset -w
# test-dep-old-hash   3   3   3   (점진적 감소)
# test-dep-new-hash   0   0   0   (점진적 증가)
```

---

## 📊 주요 컨트롤러 동작 비교

| 컨트롤러 | Watch 대상 | Desired State | Actual State | 조정 행동 |
|---------|----------|--------------|-------------|---------|
| ReplicaSet | Pod, ReplicaSet | spec.replicas | 실행 중인 파드 수 | Pod 생성/삭제 |
| Deployment | Deployment, ReplicaSet | spec.template | 현재 ReplicaSet 스펙 | RS 생성/스케일 조정 |
| DaemonSet | Node, Pod | 각 노드에 1개 | 노드별 파드 존재 여부 | 노드당 파드 생성/삭제 |
| Node | Node | kubelet 보고 상태 | 노드 Condition | Eviction, Taint 추가 |
| Endpoint | Service, Pod | Ready 파드 IP 목록 | Endpoints 오브젝트 | Endpoints 업데이트 |

---

## ⚖️ 트레이드오프

**이벤트 기반 vs 폴링 기반 Reconciliation**

쿠버네티스 컨트롤러는 기본적으로 Watch 이벤트를 받을 때 Reconcile을 실행하되, 주기적 재동기화(resync, 기본 10~30분)도 함께 수행한다. 이벤트 기반만으로는 이벤트 유실 가능성이 있기 때문이다. 재동기화는 이 안전망 역할을 한다.

**Reconciliation의 멱등성(Idempotency)**

같은 Reconcile 함수가 여러 번 호출되어도 동일한 결과여야 한다. "파드가 3개 필요한데 현재 3개다 → 아무것도 안 한다"가 올바른 동작이다. 이 원칙을 지키면 이벤트 중복 처리나 재시도가 안전해진다.

---

## 📌 핵심 정리

```
Reconciliation Loop:
  desired = etcd의 원하는 상태 (ReplicaSet spec.replicas: 3)
  actual  = 현재 실제 상태 (실행 중인 파드: 2개)
  → 차이를 감지하면 API Server를 통해 조정 행동 수행
  → 이 루프가 "자가 치유"의 실체

선언형의 의미:
  "파드 3개 만들어라" (명령형) vs "파드가 3개인 상태여야 한다" (선언형)
  선언형은 컨트롤러가 지속적으로 상태를 감시하고 유지

ownerReferences:
  파드 → ReplicaSet → Deployment 소유 관계
  상위 오브젝트 삭제 시 하위 오브젝트 연쇄 삭제(GC)
  레이블 제거로 관리 분리 가능 → 디버깅 활용

Leader Election:
  Controller Manager 다중화 시 하나만 실행
  Lease 오브젝트로 Leader 선출 및 유지
```

---

## 🤔 생각해볼 문제

**Q1.** ReplicaSet이 파드 3개를 유지하도록 설정됐는데, 누군가 직접 파드를 5개로 늘렸다. ReplicaSet Controller는 어떻게 반응하는가?

<details>
<summary>해설 보기</summary>

ReplicaSet Controller는 레이블 셀렉터와 일치하는 파드 수가 5개임을 감지하고, Desired State(3개)와 Actual State(5개)의 차이(-2)를 계산한다. 초과된 파드 2개에 DeletionTimestamp를 설정해 삭제한다. 쿠버네티스는 "항상 원하는 상태로 수렴"한다.

</details>

**Q2.** Controller Manager가 재시작됐다. Reconciliation Loop가 다시 시작될 때, 이미 실행 중인 파드들은 삭제되고 다시 생성되는가?

<details>
<summary>해설 보기</summary>

아니다. Controller Manager가 재시작되면 API Server에서 현재 상태를 다시 읽어 Actual State로 인식한다. Desired State와 일치하면 아무 작업도 하지 않는다. Reconciliation의 멱등성 덕분에 재시작이 안전하다. 이 때문에 Controller Manager를 업그레이드하거나 재시작해도 실행 중인 파드에 영향이 없다.

</details>

**Q3.** Operator 패턴이 Controller Manager와 다른 점은 무엇인가?

<details>
<summary>해설 보기</summary>

원리는 동일하다. 둘 다 Watch → Reconcile 패턴을 사용한다. 차이는 대상 리소스다. Controller Manager는 쿠버네티스 기본 리소스(Pod, ReplicaSet, Deployment 등)를 관리한다. Operator는 CRD(Custom Resource Definition)로 정의한 커스텀 리소스를 관리한다. 예를 들어 PostgreSQL Operator는 `PostgreSQLCluster`라는 CRD를 Watch해, 클러스터 상태를 선언적으로 관리한다. Reconciliation Loop를 직접 구현한다는 점이 핵심이다. 상세한 내용은 [Ch7-01. Operator 패턴](../advanced-patterns/01-operator-crd.md)에서 다룬다.

</details>

---

<div align="center">

**[⬅️ 이전: etcd와 Raft 합의 알고리즘](./03-etcd-raft.md)** | **[홈으로 🏠](../README.md)** | **[다음: kubelet — 노드 에이전트와 Probe ➡️](./05-kubelet-probes.md)**

</div>
