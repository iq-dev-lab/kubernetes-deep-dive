# 스토리지 성능 고려사항

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ReadWriteOnce, ReadWriteMany, ReadOnlyMany는 어떤 하드웨어 제약에서 나오는가?
- 로컬 PV(노드 디스크)와 원격 스토리지(EBS, NFS)의 지연 시간 차이는 실제로 얼마나 되는가?
- 데이터베이스를 쿠버네티스에서 운영할 때 관리형 RDS 대비 무엇을 감수해야 하는가?
- StorageClass의 `volumeBindingMode: WaitForFirstConsumer`가 왜 중요한가?
- IO 집약적 워크로드에서 어떤 스토리지 설계가 적합한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"쿠버네티스에 PostgreSQL을 올렸더니 성능이 너무 느리다"는 문제의 원인 대부분이 스토리지 선택 실수다. 네트워크 스토리지(NFS, EBS)를 DB 데이터 볼륨으로 사용하면 로컬 SSD 대비 지연이 10배 이상 차이 날 수 있다. WAL(Write-Ahead Log) fsync가 매 트랜잭션마다 디스크에 기록되어야 하는 RDBMS에서 이 지연은 TPS에 직접 영향을 미친다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: Postgres를 StatefulSet으로 배포, NFS PVC 사용
      TPS가 로컬 설치 대비 1/10 수준

원리를 모를 때의 판단:
  "쿠버네티스가 느린 건가?"
  → 쿠버네티스 → Docker → 직접 설치 순으로 비교 테스트 수행
  → 결과: Docker도 느림, 직접 설치는 빠름
  → "컨테이너가 느리다"는 잘못된 결론

실제 원인:
  쿠버네티스 + 로컬 SSD PVC → 직접 설치와 거의 동일한 성능
  쿠버네티스 + NFS PVC → NFS 네트워크 지연 + 동기 쓰기 제약 → 느림

핵심:
  컨테이너 자체 오버헤드는 무시할 수준
  스토리지 선택이 성능을 결정
```

---

## ✨ 올바른 접근 (After — 성능 목표에 맞는 스토리지 선택)

```
워크로드별 스토리지 선택:

고성능 읽기/쓰기 (RDBMS, WAL 집약):
  → 로컬 PV (NVMe SSD 노드 직접 마운트)
  → 또는 관리형 DB (RDS, Cloud SQL) 고려

중간 성능 (메시지 큐, 중간 규모 DB):
  → 클라우드 블록 스토리지 (EBS gp3, GCP PD SSD)
  → Provisioned IOPS 설정 (EBS io2)

공유 파일 (여러 파드에서 같은 파일 읽기):
  → ReadWriteMany 스토리지 (EFS, NFS)
  → 단, 쓰기 집약 워크로드에는 부적합

정적 콘텐츠, 읽기 전용:
  → ReadOnlyMany 가능 (configMap, 이미지 레지스트리 캐시)
```

---

## 🔬 내부 동작 원리

### AccessMode — 하드웨어 제약의 반영

```
ReadWriteOnce (RWO):
  단일 노드에서 읽기/쓰기 마운트
  구현: 블록 스토리지 (AWS EBS, GCP PD)
  제약: 블록 디바이스는 물리적으로 한 서버에만 연결됨
        (여러 서버에 연결 = SAN 구성, 대부분 지원 안 함)
  사용: StatefulSet의 DB 파드

ReadWriteMany (RWX):
  여러 노드에서 동시 읽기/쓰기
  구현: NFS 서버, AWS EFS, GlusterFS, CephFS
  제약: 쓰기 동시성 처리를 NFS 서버가 담당
        분산 락 없으면 파일 충돌 가능
  사용: 공유 로그 디렉토리, CMS 미디어 파일

ReadOnlyMany (ROX):
  여러 노드에서 읽기 전용 마운트
  구현: 블록 스토리지도 가능 (읽기만)
  사용: 공통 설정 파일, 정적 콘텐츠

ReadWriteOncePod (RWOP, 1.22+):
  단일 파드에서만 읽기/쓰기 (노드 단위보다 더 엄격)
  같은 노드의 다른 파드도 접근 불가
```

### 스토리지 계층별 지연 시간 비교

```
지연 시간 (p99 기준, 실측값 참고):

로컬 NVMe SSD:
  읽기:  0.1~0.5ms
  쓰기:  0.05~0.2ms
  fsync: 0.1~0.5ms
  → PostgreSQL 기준: 수천~수만 TPS 가능

AWS EBS gp3:
  읽기:  1~5ms
  쓰기:  1~5ms
  fsync: 1~5ms  ← WAL sync 비용
  → PostgreSQL 기준: 수백~수천 TPS

AWS EBS io2 (Provisioned IOPS):
  읽기:  0.5~2ms (iops 설정에 따라)
  쓰기:  0.5~2ms
  fsync: 0.5~2ms
  → 고성능 필요 시 비용 증가

NFS (네트워크 파일시스템):
  읽기:  2~20ms (네트워크 지연 포함)
  쓰기:  5~50ms (서버 왕복 + 동기화)
  fsync: 10~100ms  ← DB에 치명적
  → PostgreSQL: 수십~수백 TPS 수준

AWS EFS (탄력적 파일 시스템):
  지연 EBS보다 높음 (10ms+ 일반적)
  읽기/쓰기 집약 워크로드에 부적합
  공유 파일 읽기에는 허용 가능
```

### 로컬 PV (Local Persistent Volume)

노드의 실제 디스크를 직접 사용하는 PV다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-worker1
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/nvme0      # 노드의 실제 디스크 경로
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-1      # 이 노드에만 존재하는 PV
```

```
로컬 PV의 특징:
  노드에 고정됨 → Scheduler가 해당 노드에만 파드 배치
  노드 장애 시 PV 접근 불가 → 애플리케이션 레벨 HA 필요
  성능: 네트워크 스토리지 없이 직접 SSD 성능
  사용: 고성능 DB, etcd, 메시지 큐
```

### `volumeBindingMode: WaitForFirstConsumer`

```
문제 상황 (Immediate 모드, 기본값):
  PVC 생성 즉시 PV 바인딩
  → PV는 특정 가용 영역(AZ)의 디스크
  → 파드가 다른 AZ의 노드에 스케줄링되면 디스크 마운트 불가
  → MultiAttachError 또는 스케줄링 실패

WaitForFirstConsumer 해결:
  PVC 생성 → 바인딩 대기 (Pending)
  파드 생성 → Scheduler가 노드 선택
  → 선택된 노드의 AZ에 맞는 PV/디스크 생성
  → PVC 바인딩
  → 파드와 디스크가 항상 같은 AZ에 위치

StorageClass 설정:
  volumeBindingMode: WaitForFirstConsumer
  → 다중 AZ 클러스터에서 필수
```

### 데이터베이스 in-k8s vs 관리형 서비스

```
쿠버네티스에서 DB 직접 운영:
  장점:
    클러스터 내부에서 낮은 네트워크 지연
    동일한 인프라로 관리 (kubectl)
    클라우드 벤더 독립
  
  단점:
    HA 구성 복잡 (Patroni, Vitess 등 추가 도구)
    백업 자동화 직접 구성
    DB 운영 전문 지식 필요
    노드 장애 시 복구 절차 복잡
    스토리지 성능 최적화 직접 담당

관리형 서비스 (RDS, Cloud SQL, Atlas):
  장점:
    자동 HA, 자동 백업, 자동 스냅샷
    Multi-AZ failover (수십 초)
    모니터링/알람 기본 제공
    패치/업그레이드 관리형
  
  단점:
    네트워크를 통한 접근 (VPC Peering 필요)
    클라우드 벤더 종속
    비용 (EC2 직접 운영 대비 2~3배)

권장:
  OLTP DB (PostgreSQL, MySQL): 관리형 서비스 선호
    → 안정성이 중요, 운영 복잡도 감소
  캐시 (Redis), 메시지 큐 (Kafka): k8s 운영 가능
    → 데이터 손실 허용 가능, HA 패턴 비교적 단순
  OLAP / 분석 DB: 클라우드 전용 서비스 (Redshift, BigQuery)
```

---

## 💻 실전 실험

### 1. 스토리지 성능 측정 (fio)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: perf-test-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: fio-test
spec:
  containers:
  - name: fio
    image: ljishen/fio
    command: ['sh', '-c', '
      echo "=== Sequential Write ===" &&
      fio --name=seq-write --ioengine=libaio --iodepth=1
          --rw=write --bs=4k --size=1G --filename=/data/test
          --group_reporting &&
      echo "=== Random Write (fsync) ===" &&
      fio --name=rand-write --ioengine=sync --iodepth=1
          --rw=randwrite --bs=4k --size=512M --fsync=1
          --filename=/data/test2 --group_reporting
    ']
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: perf-test-pvc
EOF

kubectl logs fio-test -f
# WRITE: bw=xxx MiB/s, iops=xxx, lat (usec) avg=xxx
# 다양한 스토리지 타입으로 비교 가능
```

### 2. emptyDir(tmpfs)와 PVC 성능 비교

```bash
# emptyDir(메모리) 성능
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tmpfs-test
spec:
  containers:
  - name: fio
    image: ljishen/fio
    command: ['sh', '-c', 'fio --name=test --ioengine=sync --iodepth=1
      --rw=randwrite --bs=4k --size=256M --fsync=1
      --filename=/tmp-data/test --group_reporting']
    volumeMounts:
    - name: mem
      mountPath: /tmp-data
  volumes:
  - name: mem
    emptyDir:
      medium: Memory
EOF
```

### 3. WaitForFirstConsumer 동작 확인

```bash
# WaitForFirstConsumer StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: wait-sc
provisioner: docker.io/hostpath
volumeBindingMode: WaitForFirstConsumer
EOF

# PVC 생성 → Pending 상태
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wait-pvc
spec:
  storageClassName: wait-sc
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc wait-pvc
# STATUS: Pending  ← 파드가 사용하기 전까지 바인딩 안 됨

# 파드 생성 시 바인딩 진행
kubectl run wait-pod --image=nginx --overrides='{"spec":{"volumes":[{"name":"v","persistentVolumeClaim":{"claimName":"wait-pvc"}}],"containers":[{"name":"nginx","image":"nginx","volumeMounts":[{"name":"v","mountPath":"/data"}]}]}}'
kubectl get pvc wait-pvc  # Bound으로 변경
```

---

## 📊 스토리지 옵션 성능·기능 비교

| 스토리지 | 지연 (p99) | RWX | 노드 고정 | 주요 용도 |
|---------|---------|-----|---------|---------|
| 로컬 NVMe PV | 0.1~0.5ms | ❌ | ✅ 고정 | 고성능 DB, etcd |
| EBS gp3 | 1~5ms | ❌ | AZ 고정 | 일반 DB, 앱 데이터 |
| EBS io2 | 0.5~2ms | ❌ | AZ 고정 | 고성능 OLTP |
| NFS | 5~50ms | ✅ | ❌ | 공유 파일 |
| EFS | 10ms+ | ✅ | ❌ | 공유 파일, Lambda |
| emptyDir | ~0.1ms | ❌ (파드 내) | N/A | 임시 캐시, IPC |

---

## ⚖️ 트레이드오프

**로컬 PV의 고성능 vs 낮은 가용성**

로컬 PV는 성능이 가장 뛰어나지만, 노드에 고정되어 노드 장애 시 PV 접근이 불가능하다. PostgreSQL + Patroni처럼 애플리케이션 레벨 HA가 구현되어 있다면 로컬 PV가 최선의 선택이다. 단순히 "데이터를 잃으면 안 된다"는 이유만으로 성능이 낮은 원격 스토리지를 선택하는 것은 올바른 균형이 아니다.

**쿠버네티스 DB 운영의 진짜 비용**

컴퓨팅 비용은 클라우드 관리형 서비스보다 저렴할 수 있지만, 운영 비용(사람 시간)은 훨씬 크다. DBA 경험 없이 쿠버네티스에서 PostgreSQL Primary/Replica를 운영하면 장애 발생 시 복구에 수 시간이 걸릴 수 있다. 팀의 DB 운영 역량을 현실적으로 평가해 관리형 서비스 vs 직접 운영을 결정해야 한다.

---

## 📌 핵심 정리

```
AccessMode 선택:
  단일 노드 DB → ReadWriteOnce (EBS, 로컬 PV)
  공유 파일   → ReadWriteMany (NFS, EFS)
  읽기 전용   → ReadOnlyMany
  단일 파드 독점 → ReadWriteOncePod (1.22+)

성능 우선순위:
  로컬 NVMe > EBS io2 > EBS gp3 >> NFS/EFS

WaitForFirstConsumer:
  다중 AZ 클러스터에서 필수
  파드와 디스크가 같은 AZ에 위치 보장

DB 운영 결정 기준:
  운영 역량 있음 + 고성능 필요 → k8s + 로컬 PV + HA 솔루션
  운영 부담 줄이고 싶음 → 관리형 서비스 (RDS, Cloud SQL)
  Kafka/Redis 등 → k8s 직접 운영 합리적
```

---

## 🤔 생각해볼 문제

**Q1.** AWS EKS에서 StatefulSet PostgreSQL을 배포하는데 Multi-AZ가 필요하다. 어떤 스토리지 구성이 적합한가?

<details>
<summary>해설 보기</summary>

몇 가지 옵션이 있다. (1) EBS gp3 + WaitForFirstConsumer StorageClass: 각 파드가 같은 AZ의 EBS를 사용하고, Patroni가 AZ 간 Primary 선출을 처리한다. EBS는 AZ 내 블록 스토리지이므로 파드가 같은 AZ에 있어야 한다. (2) io2 Provisioned IOPS: 고성능 필요 시. (3) 완전 관리형 RDS Multi-AZ: k8s 외부에서 HA를 관리하므로 운영 부담이 크게 줄어든다. 대부분의 팀에는 (3)이 권장된다. 특별한 이유(벤더 종속 회피, 비용)가 있다면 (1)+(Patroni)를 고려한다.

</details>

**Q2.** Kafka를 쿠버네티스에서 운영할 때 NFS 스토리지를 사용하면 안 되는 이유는?

<details>
<summary>해설 보기</summary>

Kafka 브로커는 메시지를 로컬 디스크에 순차 쓰기로 저장하며, fsync를 통한 내구성 보장을 위해 디스크 지연이 낮아야 한다. NFS는 쓰기 지연이 10ms 이상이고, flush/fsync 성능이 매우 나쁘다. 또한 NFS는 POSIX 파일 잠금을 완전히 지원하지 않아 Kafka가 세그먼트 파일을 올바르게 관리하지 못할 수 있다. Kafka 공식 문서도 로컬 블록 스토리지(SSD/HDD)를 강력히 권장하며, NFS는 명시적으로 비권장한다.

</details>

**Q3.** 쿠버네티스 etcd 자체는 어떤 스토리지를 사용해야 하는가?

<details>
<summary>해설 보기</summary>

etcd는 Raft Log Commit에 fsync가 필수적이며, fsync 지연이 Raft 성능과 API Server 응답 속도에 직접 영향을 미친다. etcd 공식 문서는 NVMe SSD를 강력히 권장하며, p99 fsync 지연이 10ms 미만이어야 한다고 명시한다. 따라서 control-plane 노드에는 SSD가 장착된 인스턴스를 사용해야 하며, 클라우드 환경에서는 NVMe 로컬 스토리지가 있는 인스턴스 타입(AWS c5d, m5d 등)을 고려한다. HDD나 네트워크 스토리지를 etcd에 사용하면 클러스터 전반적인 응답성이 저하된다.

</details>

---

<div align="center">

**[⬅️ 이전: StatefulSet — 순서 보장과 안정적 네트워크 ID](./04-statefulset.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — Request와 Limit ➡️](../resource-scaling/01-request-limit-cgroups.md)**

</div>
