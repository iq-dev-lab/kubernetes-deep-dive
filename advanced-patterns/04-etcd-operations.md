# etcd 운영 — 백업과 성능

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `etcdctl snapshot save`로 백업하고 `snapshot restore`로 복구하는 정확한 절차는?
- etcd 클러스터에 멤버를 추가/제거할 때 어떤 순서를 지켜야 하는가?
- 디스크 I/O가 Raft Log 복제 지연을 통해 Control Plane 전체 성능을 어떻게 결정하는가?
- `etcd_disk_wal_fsync_duration_seconds`가 높으면 어떤 증상이 나타나는가?
- etcd 클러스터의 Quorum이 깨지면 어떤 일이 일어나는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

etcd는 쿠버네티스의 "유일한 진실의 원천(Source of Truth)"이다. etcd가 없으면 쿠버네티스 클러스터 전체가 동작하지 않는다. etcd 장애 시 복구 절차를 모르면 클러스터를 잃을 수 있다. 또한 etcd의 디스크 성능이 API Server 응답 속도를 직접 결정하므로, Control Plane 성능 문제의 근본 원인이 etcd인 경우가 많다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: etcd 클러스터에서 노드 하나가 죽었을 때 즉시 제거

원리를 모를 때의 판단:
  "죽은 멤버를 빨리 제거해야 한다"
  → etcdctl member remove 즉시 실행
  → 새 멤버를 같은 데이터 디렉토리로 추가 시도

실제 위험:
  3멤버 클러스터에서 1개 죽음 → 아직 Quorum(2/3) 유지
  잘못된 절차로 제거하면 Quorum 깨질 수 있음
  
  올바른 순서:
  1. 죽은 멤버 확인 (etcdctl member list)
  2. 새 멤버를 클러스터에 추가 (etcdctl member add)
     → 이때 클러스터는 4멤버로 인식 (quorum=3)
  3. 새 노드에서 etcd 시작
  4. 죽은 멤버 제거 (etcdctl member remove)
  → 항상 살아있는 멤버 수가 과반수 이상 유지
```

---

## ✨ 올바른 접근 (After — etcd 운영 절차 이해)

```
etcd 운영의 핵심 원칙:

1. 항상 홀수 개 멤버 유지 (3, 5, 7)
   짝수는 동점(split-brain) 위험
   
2. 스냅샷 백업 정기 실행
   kubectl 명령이 성공하는 동안에도 백업
   (etcd 손상 시 복구 가능)

3. fsync 지연 모니터링
   p99 > 10ms → 스토리지 교체 검토

4. 멤버 교체 절차:
   추가 → 시작 → 제거 (절대 제거 먼저 안 됨)
```

---

## 🔬 내부 동작 원리

### etcd Raft 합의 알고리즘 복습

```
3멤버 etcd 클러스터:
  Leader: etcd-1 (Write 처리)
  Follower: etcd-2, etcd-3 (Log 복제 수신)

쓰기 과정:
  1. API Server → etcd Leader에 쓰기 요청
  2. Leader → WAL(Write-Ahead Log)에 Entry 기록 (fsync)
  3. Leader → Follower들에 Entry 전송
  4. Follower → 자신의 WAL에 기록 (fsync)
  5. Follower → Leader에 ACK 전송
  6. Quorum(과반수) ACK 도착 → Leader가 Entry Commit
  7. Leader → API Server에 성공 응답
  8. API Server → 클라이언트에 응답

fsync가 성능 병목:
  WAL Entry가 디스크에 실제로 기록되었음을 보장하는 시스템콜
  HDD: 5~10ms (회전 대기)
  SATA SSD: 1~3ms
  NVMe SSD: 0.05~0.5ms
  
  p99 fsync 지연이 높으면:
    1단계 쓰기 지연 → API Server 응답 지연 → kubectl 응답 느림
    HPA Reconcile 지연 → 스케일링 반응 느림
    Controller Manager 처리 지연 → Deployment 업데이트 느림
```

### 스냅샷 백업과 복구

```bash
# 백업 (etcd 파드에서 실행)
ETCD_POD=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  snapshot save /var/lib/etcd/snapshot-$(date +%Y%m%d-%H%M%S).db

# 스냅샷 상태 확인
etcdctl snapshot status snapshot.db
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | 1234abcd |    15000 |        400 |      6.1 MB|
# +----------+----------+------------+------------+
```

```bash
# 복구 절차 (전체 클러스터 복구 시)

# 1. 모든 etcd 멤버와 API Server 중지
systemctl stop etcd
systemctl stop kube-apiserver

# 2. 스냅샷으로 복구 (각 멤버별로 별도 data-dir)
etcdctl snapshot restore snapshot.db \
  --name etcd-1 \
  --initial-cluster "etcd-1=https://192.168.1.10:2380,etcd-2=https://192.168.1.11:2380,etcd-3=https://192.168.1.12:2380" \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls https://192.168.1.10:2380 \
  --data-dir /var/lib/etcd-restored

# 3. etcd 재시작 (복구된 data-dir 사용)
# 4. API Server 재시작
# 5. 상태 확인
etcdctl member list
kubectl get nodes
```

### etcd 멤버 추가/제거

```bash
# 멤버 목록 확인
etcdctl member list
# 8e9e05c52164694d, started, etcd-1, https://192.168.1.10:2380, https://192.168.1.10:2379
# 9321abcd, started, etcd-2, ...

# 멤버 상태 확인
etcdctl endpoint status --endpoints=\
  https://192.168.1.10:2379,\
  https://192.168.1.11:2379,\
  https://192.168.1.12:2379 \
  --cluster --write-out=table

# ENDPOINT                     ID           STATUS   VERSION  IS LEADER
# https://192.168.1.10:2379   8e9e05c5...  healthy  3.5.0    true
# https://192.168.1.11:2379   9321abcd...  healthy  3.5.0    false
# https://192.168.1.12:2379   ef1234...    unreachable ...    false ← 장애

# 새 멤버 추가 (제거 전에!)
etcdctl member add etcd-4 \
  --peer-urls=https://192.168.1.13:2380

# 새 노드에서 etcd 시작 (--initial-cluster-state=existing)

# 장애 멤버 제거
etcdctl member remove ef1234...
```

### 성능 진단 메트릭

```bash
# Prometheus에서 etcd 성능 메트릭 조회 (Grafana)

# WAL fsync 지연 (핵심 지표)
etcd_disk_wal_fsync_duration_seconds{quantile="0.99"}
# 목표: < 10ms (0.01s)
# 경고: 10~100ms
# 심각: > 100ms

# DB fsync 지연
etcd_disk_backend_commit_duration_seconds{quantile="0.99"}

# Raft 제안 처리율
rate(etcd_server_proposals_committed_total[5m])

# 피어 왕복 시간 (멤버 간 네트워크)
etcd_network_peer_round_trip_time_seconds{quantile="0.99"}
# 목표: < 1ms (같은 데이터센터)

# Quorum 상태
etcd_server_has_leader  # 1이면 Leader 있음, 0이면 Quorum 없음
```

### etcd 데이터 조회 (디버깅)

```bash
# 특정 키 조회 (etcd 내 쿠버네티스 데이터)
etcdctl get /registry/pods/default/my-pod \
  --print-value-only | strings

# 파드 목록 (etcd 직접 접근)
etcdctl get /registry/pods/ --prefix --keys-only

# 전체 키 수
etcdctl get "" --prefix --keys-only | wc -l

# Compaction (오래된 리비전 정리, 공간 확보)
CURRENT_REV=$(etcdctl endpoint status --write-out=json \
  | jq .header.revision)
etcdctl compact $((CURRENT_REV - 1000))  # 최근 1000 리비전 이전 정리
etcdctl defrag  # 실제 디스크 공간 회수
```

---

## 💻 실전 실험

### 1. etcd 상태 확인 (Kind)

```bash
ETCD_POD=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')

# etcd 멤버 확인
kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# etcd 엔드포인트 상태
kubectl exec -n kube-system $ETCD_POD -- \
  etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
# https://127.0.0.1:2379 is healthy
```

### 2. 스냅샷 백업 실습

```bash
# 백업 실행
kubectl exec -n kube-system $ETCD_POD -- \
  sh -c 'ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    snapshot save /tmp/snapshot.db'

# 스냅샷 파일을 로컬로 복사
kubectl cp kube-system/$ETCD_POD:/tmp/snapshot.db ./snapshot.db

# 스냅샷 무결성 확인
ETCDCTL_API=3 etcdctl snapshot status ./snapshot.db --write-out=table
```

### 3. etcd 메트릭 확인

```bash
# etcd 메트릭 엔드포인트 접근
kubectl port-forward -n kube-system $ETCD_POD 2381:2381 &

curl http://localhost:2381/metrics | grep -E "fsync|proposals"
# etcd_disk_wal_fsync_duration_seconds_bucket{le="0.001"} 1234
# etcd_disk_wal_fsync_duration_seconds_bucket{le="0.01"} 5678
# etcd_server_proposals_committed_total 15000
```

---

## 📊 etcd 성능 기준값

| 메트릭 | 좋음 | 경고 | 심각 |
|------|-----|-----|-----|
| WAL fsync p99 | < 1ms | 1~10ms | > 10ms |
| DB commit p99 | < 25ms | 25~100ms | > 100ms |
| 피어 왕복 p99 | < 1ms | 1~10ms | > 10ms |
| 리더 변경 빈도 | < 1/시간 | 1~5/시간 | > 5/시간 |

---

## ⚖️ 트레이드오프

**멤버 수 (3 vs 5 vs 7)**

멤버가 많을수록 내결함성이 높아지지만(5멤버=2개 장애 허용), 쓰기 성능이 저하된다. 모든 쓰기에 Quorum ACK가 필요하므로, 멤버가 많을수록 ACK 수집에 시간이 걸린다. 프로덕션은 3멤버(1개 장애 허용)가 일반적이고, 고가용성이 중요한 경우 5멤버를 사용한다. 7멤버 이상은 쓰기 성능 저하가 크므로 권장하지 않는다.

**Compaction 주기**

etcd는 모든 쓰기를 리비전으로 관리하므로, Compaction 없이는 디스크 사용량이 계속 증가한다. 자주 Compaction하면 디스크를 절약하지만 CPU와 I/O 부하가 증가한다. `--auto-compaction-retention=8h`로 8시간마다 자동 Compaction을 설정하는 것이 일반적이다.

---

## 📌 핵심 정리

```
etcd 핵심 역할:
  쿠버네티스의 유일한 데이터 저장소
  모든 오브젝트 (Pod, Service, Deployment...)

fsync 지연 → Control Plane 성능 직결:
  WAL fsync p99 > 10ms → kubectl 지연, API Server 느림
  해결: NVMe SSD, 전용 디스크, 빠른 I/O 노드

백업:
  etcdctl snapshot save → .db 파일
  주기적 백업 (크론잡으로 자동화)
  etcdctl snapshot restore로 복구

멤버 교체 순서 (중요!):
  새 멤버 추가(etcdctl member add) → 시작 → 죽은 멤버 제거
  절대 제거 먼저 하면 안 됨 (Quorum 손실 위험)

Quorum:
  3멤버 → 2개 필요, 1개 장애 허용
  quorum 손실 → etcd Read-only, 클러스터 작동 중단
```

---

## 🤔 생각해볼 문제

**Q1.** etcd 스냅샷을 복구할 때 `--initial-cluster-token`을 변경해야 하는 이유는?

<details>
<summary>해설 보기</summary>

기존 클러스터와 복구된 클러스터가 같은 token을 사용하면, 네트워크 상에서 두 클러스터가 서로를 인식해 Raft 메시지가 섞일 수 있다. 다른 token을 사용하면 복구된 클러스터가 완전히 새로운 독립적인 클러스터로 인식되어, 기존 클러스터와 통신하지 않는다. 재해 복구 시나리오에서 기존 클러스터가 네트워크에 있는 경우 특히 중요하다.

</details>

**Q2.** etcd WAL과 snapshots 디렉토리가 같은 디스크에 있는 경우 어떤 문제가 생길 수 있는가?

<details>
<summary>해설 보기</summary>

etcd snapshot 저장 시 디스크 I/O가 일시적으로 증가하면 WAL fsync 지연이 증가한다. WAL fsync 지연이 높아지면 Raft 합의 속도가 느려지고, API Server 응답이 지연된다. 프로덕션 권장사항은 WAL과 데이터(snapshots)를 서로 다른 SSD에 두는 것이다(`--wal-dir`로 별도 경로 지정). 외부 백업은 NFS나 S3에 저장해 etcd 디스크 I/O와 완전히 분리하는 것이 좋다.

</details>

**Q3.** etcd 클러스터의 리더가 자주 바뀌면 어떤 증상이 나타나는가?

<details>
<summary>해설 보기</summary>

리더 선출 중에는 쓰기 요청이 처리되지 않으므로, 리더 변경 빈도가 높으면 API Server에서 간헐적인 타임아웃이 발생한다. `kubectl get pods`가 갑자기 오래 걸리거나 실패하는 증상이 나타난다. 원인은 주로 (1) 멤버 간 네트워크 지연이 높아 heartbeat 타임아웃이 잦음, (2) 리더 노드의 디스크 I/O가 느려 heartbeat 응답이 지연됨, (3) 리더 노드의 CPU 과부하로 heartbeat 처리 지연이다. `etcd_server_leader_changes_seen_total` 메트릭으로 확인한다.

</details>

---

<div align="center">

**[⬅️ 이전: Service Mesh — Istio와 Envoy Sidecar](./03-service-mesh-istio.md)** | **[홈으로 🏠](../README.md)** | **[다음: 멀티 클러스터 운영 — 네트워킹과 트래픽 라우팅 ➡️](./05-multi-cluster.md)**

</div>
