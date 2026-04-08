# PV와 PVC — 동적 프로비저닝

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- PV, PVC, StorageClass 세 오브젝트의 관계는 무엇이고, 각각 누가 관리하는가?
- 동적 프로비저닝에서 클라우드 디스크는 어느 시점에 생성되는가?
- PV-PVC 바인딩 알고리즘은 어떤 기준으로 매칭하는가?
- ReclaimPolicy(Retain/Delete)는 PVC 삭제 시 데이터에 어떤 영향을 미치는가?
- PVC가 Pending 상태에서 멈출 때 어떻게 진단하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"PVC를 삭제했더니 DB 데이터가 날아갔다"는 실수가 자주 발생한다. Delete ReclaimPolicy의 기본 동작을 모르면 PVC 삭제 한 번으로 수 기가바이트 데이터가 복구 불가능하게 삭제된다. 반대로 Retain을 설정하면 PVC를 삭제해도 PV와 실제 디스크가 유지되므로 데이터를 복구할 수 있다.

또한 PVC Pending 진단은 매우 흔한 운영 이슈다. StorageClass 이름 불일치, AccessMode 불일치, 스토리지 용량 부족 — 이 중 무엇이 원인인지 빠르게 찾으려면 바인딩 알고리즘을 이해해야 한다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 스테이징 환경 정리 중 kubectl delete pvc --all 실행
      → DB 데이터 전체 삭제

원리를 모를 때의 판단:
  "PVC 삭제 = PVC 오브젝트만 삭제" 라고 생각
  
실제 동작:
  PVC 삭제
  → PV 상태 Released
  → StorageClass의 ReclaimPolicy: Delete
  → CSI 드라이버가 실제 AWS EBS 볼륨 DeleteVolume() 호출
  → 디스크 내용 영구 삭제 (복구 불가)

올바른 운영:
  중요 PVC에는 finalizer 확인 또는
  StorageClass에 reclaimPolicy: Retain 사용
  PV 삭제 전 반드시 데이터 백업 확인
```

---

## ✨ 올바른 접근 (After — PV/PVC 수명주기를 이해한 운영)

```
환경별 ReclaimPolicy 전략:
  개발/테스트: reclaimPolicy: Delete (자동 정리, 비용 절감)
  스테이징:    reclaimPolicy: Retain (실수 방지, 수동 정리)
  프로덕션:    reclaimPolicy: Retain + 별도 백업 정책

데이터 보호 레이어:
  1. Retain ReclaimPolicy → PVC 삭제해도 디스크 유지
  2. VolumeSnapshot → 정기 스냅샷 (AWS EBS Snapshot)
  3. 애플리케이션 레벨 백업 → pg_dump, mysqldump
```

---

## 🔬 내부 동작 원리

### PV / PVC / StorageClass 관계

```
StorageClass (관리자가 정의)
  → "이 클래스를 사용하면 이런 종류의 디스크를 만들어라"
  → CSI 드라이버 지정, 파라미터(디스크 타입, IOPS), ReclaimPolicy

PVC (개발자/사용자가 요청)
  → "나는 10GB ReadWriteOnce 스토리지가 필요하다"
  → storageClassName 지정 또는 기본 StorageClass 사용

PV (클러스터가 자동 생성 또는 관리자가 사전 생성)
  → "10GB EBS gp3 볼륨 vol-0abc123이 있다"
  → PVC와 1:1 바인딩

관계도:
  StorageClass ─── (동적 프로비저닝 트리거) ──► PV 자동 생성
       ↑                                              │
       │ storageClassName 참조               1:1 바인딩
       │                                              │
      PVC ◄────────────────────────────────────────────┘
       │
       └──► 파드의 spec.volumes[].persistentVolumeClaim.claimName
```

### 동적 프로비저닝 전체 흐름

```
kubectl apply -f pvc.yaml
      │
      ▼
1. PVC 생성 (API Server → etcd)
   spec.storageClassName: standard
   spec.accessModes: [ReadWriteOnce]
   spec.resources.requests.storage: 10Gi
   status.phase: Pending

      │ PVC Watch 이벤트 (ADDED)
      ▼
2. PersistentVolume Controller (Controller Manager)
   StorageClass 조회 → provisioner: kubernetes.io/aws-ebs
   → External Provisioner(CSI 드라이버)에 CreateVolume 요청

      │
      ▼
3. CSI 드라이버 (aws-ebs-csi-driver)
   AWS API 호출: ec2.CreateVolume()
   → EBS 볼륨 생성 (vol-0abc123, 10GB, gp3)
   → 완료까지 5~30초 소요

      │
      ▼
4. PV 자동 생성
   spec.capacity.storage: 10Gi
   spec.accessModes: [ReadWriteOnce]
   spec.csi.volumeHandle: vol-0abc123
   spec.persistentVolumeReclaimPolicy: Delete
   spec.storageClassName: standard

      │
      ▼
5. PV-PVC 바인딩
   PVC.spec.volumeName = pv-abc123
   PV.spec.claimRef = {name: my-pvc, namespace: default}
   양쪽 status.phase: Bound

      │
      ▼
6. 파드가 PVC 사용 시
   kubelet → CSI Node Plugin 호출
   ControllerPublishVolume() → EBS를 EC2 인스턴스에 Attach
   NodeStageVolume()         → 파일시스템 포맷/마운트
   NodePublishVolume()       → 파드 경로에 bind mount
```

### PV-PVC 바인딩 알고리즘

PVC는 조건을 만족하는 PV를 찾아 바인딩된다.

```
매칭 조건 (모두 충족해야 바인딩):

  1. storageClassName 일치
     PVC.spec.storageClassName == PV.spec.storageClassName
     (또는 둘 다 비어있음)

  2. AccessMode 포함
     PVC.spec.accessModes ⊆ PV.spec.accessModes
     (PV가 PVC가 요청한 모드를 모두 지원해야 함)

  3. 용량 충족
     PV.spec.capacity.storage >= PVC.spec.resources.requests.storage
     (PV가 PVC 요청 용량 이상이어야 함)

  4. 레이블 셀렉터 (optional)
     PVC.spec.selector가 있으면 PV 레이블과 매칭

  5. VolumeMode 일치
     Filesystem vs Block (미지정 시 Filesystem)

바인딩 우선순위:
  조건 만족하는 PV 중 용량이 가장 작은 PV 선택
  (불필요하게 큰 PV를 낭비하지 않기 위해)
```

### ReclaimPolicy 동작

```
PVC 삭제 시 시나리오:

  Delete (기본값, 동적 프로비저닝):
    PVC 삭제 → PV Released → CSI DeleteVolume() 호출
    → 실제 디스크 삭제 (영구적)
    → kubectl get pv → PV도 사라짐

  Retain:
    PVC 삭제 → PV Released (그러나 데이터 유지)
    → 관리자가 수동으로 PV.spec.claimRef 제거 후 재사용 가능
    → kubectl get pv → PV가 Released 상태로 남음
    → 실제 디스크는 유지됨

  데이터 복구 (Retain 사용 시):
    1. 새 PVC 생성
    2. PV.spec.claimRef를 새 PVC로 수동 설정
    3. 바인딩 완료 → 데이터 접근 가능
```

---

## 💻 실전 실험

### 1. StorageClass 확인

```bash
# 현재 클러스터의 StorageClass 목록
kubectl get storageclass
# NAME                 PROVISIONER                RECLAIMPOLICY
# standard (default)  docker.io/hostpath          Delete
# (Kind 클러스터의 기본 StorageClass)

# StorageClass 상세
kubectl describe storageclass standard
```

### 2. 동적 프로비저닝 실습

```bash
# PVC 생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
EOF

# PVC → PV 자동 생성 관찰
kubectl get pvc my-pvc -w
# NAME     STATUS    VOLUME   CAPACITY   ...
# my-pvc   Pending   ...      ...        ← 프로비저닝 중
# my-pvc   Bound     pvc-xxx  1Gi        ← 바인딩 완료

# 자동 생성된 PV 확인
kubectl get pv
kubectl describe pv $(kubectl get pvc my-pvc -o jsonpath='{.spec.volumeName}')
```

### 3. PVC Pending 원인 진단

```bash
# 존재하지 않는 StorageClass 지정 → Pending
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broken-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
  storageClassName: non-existent-class  # 없는 StorageClass
EOF

kubectl describe pvc broken-pvc | grep -A5 "Events:"
# Warning  ProvisioningFailed  storageclass.storage.k8s.io "non-existent-class" not found
```

### 4. PVC 삭제 시 ReclaimPolicy 동작 관찰

```bash
# PVC 삭제 전 PV 이름 기록
PV_NAME=$(kubectl get pvc my-pvc -o jsonpath='{.spec.volumeName}')
echo $PV_NAME

# PVC 삭제
kubectl delete pvc my-pvc

# PV 상태 변화 확인
kubectl get pv $PV_NAME
# (Delete 정책): PV가 사라짐
# (Retain 정책): PV가 Released 상태로 남음
```

### 5. 파드에서 PVC 사용

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pvc-user
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "persistent data" > /data/file.txt && sleep 3600']
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
EOF

# 파드가 Running이 된 후 데이터 확인
kubectl exec pvc-user -- cat /data/file.txt
# persistent data

# 파드 삭제 후 재생성해도 데이터 유지
kubectl delete pod pvc-user
kubectl apply -f - <<< "..." # 동일 PVC 재사용 파드 재생성
kubectl exec pvc-user -- cat /data/file.txt
# persistent data  ← PVC가 데이터를 유지
```

---

## 📊 PV 수명주기 상태 비교

| 상태 | 의미 | 전환 조건 |
|-----|-----|---------|
| Available | PV 생성, PVC 미바인딩 | 정적 프로비저닝 후 기본 상태 |
| Bound | PVC와 바인딩 완료 | PVC 생성 + 조건 매칭 |
| Released | PVC 삭제, PV는 유지 (Retain) | PVC 삭제 시 |
| Failed | 자동 회수 실패 | Recycle 정책 오류 |

---

## ⚖️ 트레이드오프

**동적 프로비저닝 vs 정적 프로비저닝**

동적 프로비저닝은 PVC 생성만으로 디스크가 자동 생성되어 편리하다. 그러나 PVC/파드 실수 삭제 시 데이터도 사라질 수 있다(Delete 정책). 정적 프로비저닝은 관리자가 사전에 PV를 수동 생성하므로 제어가 명확하지만, 운영 부담이 크다. 프로덕션에서는 동적 프로비저닝 + Retain 정책의 조합이 편리함과 안전성을 균형있게 유지한다.

**AccessMode 제약**

ReadWriteOnce(RWO)는 한 노드에서만 마운트 가능하다. Deployment 파드가 롤링 업데이트 중 두 노드에 파드가 동시에 존재할 때, 기존 파드 노드와 신규 파드 노드가 다르면 RWO 볼륨 Attach 충돌이 발생할 수 있다. 이 경우 `strategy.type: Recreate`로 전략을 바꾸거나 ReadWriteMany(RWX)를 지원하는 스토리지(NFS, EFS)를 사용해야 한다.

---

## 📌 핵심 정리

```
세 오브젝트 역할:
  StorageClass  → "어떤 종류의 디스크를 어떻게 만들지" 정의 (관리자)
  PVC           → "나는 이런 스토리지가 필요하다" 요청 (개발자)
  PV            → "이 디스크가 있다" 실체 (자동 생성 또는 관리자)

동적 프로비저닝 순서:
  PVC 생성 → Controller가 CSI 호출 → 실제 디스크 생성
  → PV 자동 생성 → PVC-PV 바인딩 → 파드 마운트

ReclaimPolicy:
  Delete (기본): PVC 삭제 → 디스크 삭제 (데이터 영구 소실)
  Retain:        PVC 삭제 → 디스크 유지 (수동 재연결 가능)
  프로덕션: Retain 강력 권장

PVC Pending 진단:
  kubectl describe pvc → Events 섹션
  - StorageClass 없음 → storageClassName 확인
  - AccessMode 불일치 → PV spec 확인
  - 용량 부족 → PV 용량 vs PVC 요청 확인
```

---

## 🤔 생각해볼 문제

**Q1.** PVC가 Bound 상태인데 파드가 해당 PVC를 사용하려 할 때 `Multi-Attach error`가 발생했다. 원인과 해결책은?

<details>
<summary>해설 보기</summary>

ReadWriteOnce PV를 두 개의 파드가 동시에 마운트하려 할 때 발생한다. 가장 흔한 케이스는 Deployment 롤링 업데이트 중 신규 파드와 기존 파드가 다른 노드에 있는 경우다. 해결책은 세 가지다. (1) `strategy.type: Recreate`로 기존 파드를 먼저 종료 후 신규 파드 생성. (2) `podAffinityRequired`로 항상 같은 노드에 배치. (3) ReadWriteMany를 지원하는 스토리지(NFS, AWS EFS)로 변경. StatefulSet에서는 각 파드가 독립 PVC를 가지므로 이 문제가 발생하지 않는다.

</details>

**Q2.** 정적으로 생성한 PV(관리자가 수동 생성)를 특정 PVC에만 바인딩되도록 예약하는 방법은?

<details>
<summary>해설 보기</summary>

PV에 `spec.claimRef`를 미리 설정하면 해당 PVC에만 바인딩된다. `claimRef.name`과 `claimRef.namespace`를 지정하면 다른 PVC가 이 PV를 가져가지 못한다. 반대로 PVC에 `spec.volumeName`을 지정하면 해당 PV에만 바인딩된다. 이 두 가지를 함께 사용하면 PV-PVC를 명시적으로 1:1 예약할 수 있다.

</details>

**Q3.** StorageClass의 `allowVolumeExpansion: true` 설정은 무슨 의미인가?

<details>
<summary>해설 보기</summary>

PVC의 용량 요청을 늘릴 수 있게 허용하는 설정이다. 이 설정이 활성화된 StorageClass를 사용하면, `kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'`처럼 용량 확장이 가능하다. CSI 드라이버가 실제 디스크를 온라인으로 확장하고, 파드가 재시작 없이 또는 재시작 후 파일시스템이 확장된다. 단, 축소(용량 줄이기)는 지원하지 않는다.

</details>

---

> ⬅️ 이전: [01. Volume과 파드 — emptyDir부터 configMap까지](./01-volumes.md)  
> ➡️ 다음: [03. CSI — 스토리지 드라이버 표준 인터페이스](./03-csi.md)
