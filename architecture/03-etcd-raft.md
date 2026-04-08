# etcd와 Raft 합의 알고리즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- etcd가 클러스터 상태의 "단일 진실 공급원"인 이유는 무엇인가?
- Raft의 Leader Election은 어떻게 이루어지고, 네트워크 분리 시 어떤 일이 벌어지는가?
- Quorum(과반수)이 없으면 etcd는 왜 쓰기를 거부하는가?
- etcd의 디스크 I/O가 Control Plane 전체 성능에 어떤 영향을 미치는가?
- `etcdctl`로 클러스터 상태를 직접 조회하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

etcd는 쿠버네티스에서 가장 중요하고 가장 취약한 컴포넌트다. API Server, Scheduler, Controller Manager는 Stateless하게 재시작이 가능하지만, etcd를 잃으면 클러스터의 모든 상태 정보가 사라진다. 어떤 파드가 어떤 노드에 있는지, 어떤 Service가 있는지, 어떤 Secret이 설정됐는지 모두 etcd에만 존재한다.

또한 etcd의 I/O 지연은 직접 API Server 응답 속도로 이어진다. "kubectl이 느리다"는 현상의 상당수가 etcd 디스크 성능 문제다. Raft 합의에 디스크 fsync가 필수적으로 포함되기 때문이다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 3-node etcd 클러스터에서 노드 1개가 다운됨

원리를 모를 때의 판단:
  "노드 하나 다운됐지만 2개가 살아있으니 괜찮겠지"
  → 그냥 방치

실제 상황:
  3노드 중 1개 다운 → Quorum = 2 필요 → 2개 생존 → 쓰기 가능 (정상)

그러나:
  3노드 중 2개 다운 → Quorum = 2 필요 → 1개만 생존 → 쓰기 불가
  → kubectl apply, kubectl delete 모두 실패
  → "API Server가 죽었나?" 혼동
  → API Server 재시작 시도 → 당연히 해결 안 됨

원리를 알았다면:
  etcd 상태 먼저 확인 → 죽은 멤버 제거 후 새 멤버 추가 → 복구
```

---

## ✨ 올바른 접근 (After — Raft를 알고 난 운영)

```
etcd 클러스터 노드 수 설계:
  1개 → 개발/테스트 용도. 단일 장애 지점. 프로덕션 절대 불가
  3개 → 1개 장애 허용 (Quorum: 2). 최소 권장 프로덕션 구성
  5개 → 2개 장애 허용 (Quorum: 3). 큰 클러스터 권장
  7개 → 3개 장애 허용. 성능 저하 시작. 일반적으로 비권장

Quorum = (N / 2) + 1
  N=3 → Quorum=2 → 1개까지 장애 허용
  N=5 → Quorum=3 → 2개까지 장애 허용
  짝수 노드(4, 6)는 홀수 노드보다 장애 허용 수가 같으면서 복잡도만 높아짐 → 비권장
```

---

## 🔬 내부 동작 원리

### etcd의 역할과 저장 구조

etcd는 쿠버네티스의 모든 상태를 키-값 형식으로 저장한다. 모든 쿠버네티스 오브젝트는 `/registry/<resource>/<namespace>/<name>` 경로에 protobuf 형식으로 저장된다.

```
/registry/pods/default/my-app-xxx
/registry/deployments/default/my-deployment
/registry/services/default/my-service
/registry/secrets/default/my-secret
/registry/configmaps/kube-system/coredns
/registry/nodes/worker-1
```

API Server만 etcd에 직접 접근한다. 다른 모든 컴포넌트는 API Server의 REST API를 통해 간접적으로 읽고 쓴다.

### Raft 합의 알고리즘

etcd가 여러 노드에 걸쳐 데이터 일관성을 보장하는 방법이다.

**Leader Election**

클러스터의 모든 노드는 세 가지 역할 중 하나다: Leader, Follower, Candidate.

```
초기 상태: 모든 노드가 Follower
  │
  ▼ Election Timeout (랜덤 150~300ms) 만료 시
Follower → Candidate (자신에게 투표, term 증가)
  │
  ▼ 다른 노드에 RequestVote 전송
  │  과반수(Quorum)로부터 투표 받으면
Candidate → Leader
  │
  ▼ Leader는 주기적으로 Heartbeat(AppendEntries) 전송
  │  Follower들이 Heartbeat를 받으면 Election Timeout 리셋
  │
  ▼ Leader 장애 시
Heartbeat 없음 → Follower Election Timeout 만료 → 새 Election 시작
```

Election Timeout이 랜덤인 이유: 모든 Follower가 동시에 Candidate가 되어 투표가 분산되는 "split vote" 상황을 방지한다.

**Log Replication**

쓰기 요청은 반드시 Leader를 통해 이루어진다.

```
클라이언트(API Server) → Leader에게 쓰기 요청
  │
  ▼ Leader가 Log Entry를 로컬에 추가
  │
  ▼ Follower들에게 AppendEntries RPC 전송
  │
  ▼ Quorum(과반수)이 "Log Entry 받았다" 응답
  │
  ▼ Leader가 Entry를 "Committed"로 표시
  │  디스크에 fsync → 이것이 etcd I/O 지연의 원인
  │
  ▼ 클라이언트에게 성공 응답
  │
  ▼ 다음 Heartbeat에서 Follower들도 Commit 처리
```

**왜 Quorum(과반수) 응답이 필요한가**

Quorum 없이 Leader만 기록하면 Leader 장애 시 데이터가 유실될 수 있다. Quorum이 확인된 데이터는, 새 Leader가 선출될 때 반드시 해당 데이터를 가진 노드가 Leader가 되도록 Raft 알고리즘이 보장한다.

```
3노드 클러스터, Quorum=2인 경우:
  쓰기 성공 조건: Leader + 1개 Follower가 기록
  → Leader가 죽어도 데이터를 가진 Follower가 새 Leader가 됨 → 데이터 안전

쓰기 실패 시나리오:
  3노드 중 2개 다운 → 남은 1개 노드만으로 Quorum(2) 불충족
  → 이 노드가 Leader여도 쓰기 거부
  → 이유: 과반수 없이 기록하면 나중에 분리됐던 2개 노드가 복구됐을 때 데이터 충돌 가능
```

**네트워크 파티션(Split-Brain) 방지**

```
네트워크 분리로 5노드 클러스터가 3/2로 분리된 경우:

  그룹 A (3노드): Quorum=3 충족 → 쓰기 가능 → 계속 서비스
  그룹 B (2노드): Quorum=3 미충족 → 쓰기 거부 → 읽기는 가능하지만 stale 데이터

  네트워크 복구 후:
  그룹 B가 그룹 A의 최신 로그를 따라잡음 (Log Replication)
  그룹 B에서 발생한 쓰기 시도는 이미 거부됐으므로 충돌 없음
```

이것이 Raft가 Split-Brain을 방지하는 방법이다. 과반수가 없는 파티션은 쓰기 자체를 거부해 두 파티션이 서로 다른 상태로 발전하는 것을 막는다.

### etcd I/O와 Control Plane 성능

Raft의 Log Commit에는 반드시 `fsync`가 필요하다. 메모리 버퍼에 쓰는 것으로는 충분하지 않다. 노드 재시작 시 로그 복구를 위해 디스크에 영속적으로 기록해야 한다.

```
쓰기 지연 = 네트워크 RTT (Leader → Follower) + 디스크 fsync 시간

SSD (NVMe): fsync ≈ 0.1~1ms    → etcd 쓰기 지연 1~5ms
HDD:        fsync ≈ 5~20ms     → etcd 쓰기 지연 10~30ms
네트워크 디스크: fsync ≈ 불규칙 → 권장하지 않음

etcd 권장 디스크: NVMe SSD
etcd 권장 디스크 지연: p99 < 10ms (etcd_disk_wal_fsync_duration_seconds)
```

`kubectl apply`가 느리다면 etcd 디스크 I/O를 먼저 확인해야 한다.

---

## 💻 실전 실험

### 1. etcdctl로 클러스터 상태 직접 조회 (Kind 클러스터)

```bash
# etcd 파드 이름 확인
ETCD_POD=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')

# etcd 클러스터 상태 확인
kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  endpoint status --write-out=table

# 출력:
# ENDPOINT             ID               IS LEADER  REVISION  ...
# https://127.0.0.1:2379  8e9e05c52164694d  true       1234      ...
```

### 2. etcd에서 파드 원본 데이터 조회

```bash
# 파드 키 목록 조회
kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  get /registry/pods/default --prefix --keys-only

# 출력:
# /registry/pods/default/my-app-xxx
# /registry/pods/default/test-pod

# 특정 파드 데이터 조회 (protobuf 형식, 읽기 어려움)
kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  get /registry/pods/default/test-pod | strings | head -30
```

### 3. Raft 리더 확인

```bash
# 어느 노드가 현재 Leader인지 확인
kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  endpoint status --write-out=json | python3 -m json.tool | grep -A3 "isLeader"
```

### 4. etcd 디스크 I/O 지연 측정

```bash
# etcd 메트릭으로 fsync 지연 확인
kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  check perf

# Prometheus 메트릭으로 확인 (Prometheus가 있는 경우)
# etcd_disk_wal_fsync_duration_seconds_bucket
# etcd_disk_backend_commit_duration_seconds_bucket
```

### 5. etcd 스냅샷 백업 (운영 필수)

```bash
# etcd 스냅샷 생성
kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup.db

# 파드 밖으로 복사
kubectl cp kube-system/$ETCD_POD:/tmp/etcd-backup.db ./etcd-backup.db

# 스냅샷 상태 확인
etcdctl snapshot status etcd-backup.db --write-out=table
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | fe01cf57 |       10 |          7 | 2.1 MB     |
```

---

## 📊 etcd 클러스터 구성별 비교

| 노드 수 | Quorum | 장애 허용 | 쓰기 지연 | 권장 용도 |
|--------|--------|---------|---------|---------|
| 1 | 1 | 0개 | 최소 | 개발/테스트 전용 |
| 3 | 2 | 1개 | 낮음 | 프로덕션 최소 구성 |
| 5 | 3 | 2개 | 중간 | 큰 클러스터 권장 |
| 7 | 4 | 3개 | 높음 | 특수한 고가용성 요구사항 |

---

## ⚖️ 트레이드오프

**일관성 vs 가용성**

etcd는 Quorum이 없으면 쓰기를 거부한다. 이는 CAP 정리에서 CP(Consistency + Partition tolerance)를 선택한 것이다. 과반수 노드 장애 시 쓰기 가용성을 희생해 데이터 일관성을 보장한다.

읽기는 기본적으로 Leader에서만 수행한다(linearizable read). `--consistency=s` 옵션으로 Follower에서 읽을 수 있지만, 약간의 stale 데이터를 허용하게 된다.

**etcd 노드 수 증가의 단점**

노드가 많을수록 Quorum이 높아지고, Quorum에 도달하는 데 더 많은 네트워크 왕복이 필요하다. 5노드에서 7노드로 늘리면 장애 허용은 2→3으로 1개 늘지만, 쓰기 지연은 증가한다. 일반적으로 5노드가 최적의 균형점이다.

---

## 📌 핵심 정리

```
etcd = 쿠버네티스의 단일 진실 공급원
  모든 클러스터 상태가 저장됨 (파드, 서비스, 시크릿 등)
  API Server만 직접 접근 가능
  손실 시 클러스터 상태 전부 소실 → 정기 백업 필수

Raft 합의:
  Leader가 쓰기를 받아 Follower에게 복제
  Quorum(N/2+1) 응답 후 Commit → fsync → 클라이언트 응답
  과반수 없으면 쓰기 거부 → Split-Brain 방지

Quorum 공식:
  N=3 → Q=2 → 1개 장애 허용 (프로덕션 최소)
  N=5 → Q=3 → 2개 장애 허용 (권장)

성능 핵심:
  etcd I/O 지연 = Control Plane 응답 지연
  NVMe SSD 권장, fsync p99 < 10ms 목표
  HDD나 네트워크 디스크 → 심각한 성능 저하
```

---

## 🤔 생각해볼 문제

**Q1.** 5노드 etcd 클러스터에서 동시에 3개 노드가 다운됐다. 남은 2노드에서 `kubectl get pods`는 동작하는가?

<details>
<summary>해설 보기</summary>

`kubectl get pods`의 읽기 요청은 상황에 따라 다르다. etcd의 기본 읽기 모드(linearizable)는 Leader를 통해 처리되는데, Quorum(3)이 없으므로 Leader가 선출되지 않는다. 따라서 API Server도 etcd에서 데이터를 읽지 못하고 읽기도 실패한다. 단, API Server의 watch cache에 이미 올라온 데이터는 캐시에서 응답할 수 있는 경우가 있다. `--consistency=s`(직렬성 완화) 모드라면 Follower에서 stale 데이터를 읽을 수 있지만, 이것이 활성화되어 있지 않은 것이 일반적이다.

</details>

**Q2.** etcd 데이터를 암호화하지 않으면 어떤 위험이 있는가?

<details>
<summary>해설 보기</summary>

etcd에는 Secret 오브젝트도 저장된다. Secret의 값은 Base64 인코딩일 뿐 암호화가 아니므로, etcd 파일에 직접 접근하면 Secret 값이 평문으로 노출된다. 이를 막기 위해 API Server의 `--encryption-provider-config` 옵션으로 etcd 저장 시 AES-GCM 또는 AES-CBC 암호화를 적용할 수 있다. `kubectl get secret my-secret -o yaml`에서 Base64로 보이는 값은 etcd에서도 암호화된 상태이므로 파일 직접 접근으로는 읽을 수 없게 된다.

</details>

**Q3.** `kubectl apply`가 갑자기 10초 이상 걸린다. 어떤 순서로 진단하는가?

<details>
<summary>해설 보기</summary>

1. API Server 응답 지연 여부 확인: `time kubectl get pods`로 기본 조회 지연 측정. 2. etcd 디스크 지연 확인: `etcdctl check perf` 또는 `etcd_disk_wal_fsync_duration_seconds` 메트릭. 3. Admission Webhook 지연 확인: `kubectl get mutatingwebhookconfigurations`로 타임아웃 설정 확인. Webhook 서버 응답이 느리면 모든 리소스 생성이 느려진다. 4. API Server 로그에서 slow request 확인.

</details>

---

> ⬅️ 이전: [02. API Server](./02-api-server.md)  
> ➡️ 다음: [04. Controller Manager — Reconciliation Loop의 실체](./04-controller-manager-reconciliation.md)
