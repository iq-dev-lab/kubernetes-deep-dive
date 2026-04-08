# CSI — 스토리지 드라이버 표준 인터페이스

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CSI 등장 이전 in-tree 플러그인 방식의 문제점은 무엇이었는가?
- Controller Plugin과 Node Plugin이 어떤 역할을 분담하는가?
- kubelet이 CSI Node Plugin을 호출해 볼륨을 마운트하는 단계는?
- CSI Driver가 어떤 kuberntees 오브젝트로 배포되는가?
- `VolumeAttachment` 오브젝트는 언제 생성되고 무슨 역할을 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

스토리지 문제는 "파드가 Pending인데 PVC Attach 실패"처럼 나타난다. CSI 구조를 이해하면 Controller Plugin이 문제인지(볼륨 생성/Attach), Node Plugin이 문제인지(마운트)를 구분해 진단할 수 있다. 또한 EBS, EFS, NFS 같은 다양한 스토리지를 쿠버네티스에 연결할 때 CSI 드라이버가 무엇을 하는지 알아야 올바른 StorageClass 파라미터를 설정할 수 있다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 파드가 ContainerCreating에서 멈춤
      kubectl describe pod → "Unable to attach or mount volumes"

원리를 모를 때의 판단:
  "PVC 문제인가?" → kubectl get pvc → Bound 상태 (정상)
  "PV 문제인가?" → kubectl get pv → Bound 상태 (정상)
  → 원인 모름

CSI 구조 알면 진단 경로:
  볼륨 Attach 실패 (노드에 디스크 연결 안 됨):
    kubectl get volumeattachment   → Attached: false?
    CSI Controller Plugin 로그 확인
    → AWS EBS: IAM 권한 부족? 볼륨 가용영역 다름?

  볼륨 Mount 실패 (디스크는 연결됐지만 파일시스템 마운트 안 됨):
    CSI Node Plugin 로그 확인
    → 파일시스템 손상? 권한 문제?
```

---

## ✨ 올바른 접근 (After — CSI 계층을 이해한 진단)

```
볼륨 마운트 3단계:
  Provision → Attach → Mount

  Provision (볼륨 생성):
    CSI Controller Plugin → CreateVolume()
    AWS: EBS 볼륨 생성
    진단: kubectl describe pvc → Events

  Attach (노드에 연결):
    CSI Controller Plugin → ControllerPublishVolume()
    AWS: EBS를 EC2 인스턴스에 Attach
    진단: kubectl get volumeattachment

  Mount (파드 경로에 마운트):
    CSI Node Plugin → NodeStageVolume() + NodePublishVolume()
    파일시스템 마운트 → bind mount
    진단: kubelet 로그, CSI Node Pod 로그
```

---

## 🔬 내부 동작 원리

### in-tree 방식의 문제

CSI 이전에는 스토리지 드라이버가 쿠버네티스 코어 코드에 포함(in-tree)되어 있었다.

```
in-tree 방식의 문제점:

  1. 쿠버네티스 릴리즈 의존성
     AWS EBS 드라이버를 수정하려면 쿠버네티스 전체 릴리즈 필요
     → AWS가 k8s 릴리즈 일정에 종속

  2. 모든 드라이버 코드가 kubelet에 포함
     사용하지 않는 스토리지 드라이버 코드도 kubelet 바이너리에 포함
     → 바이너리 크기 증가, 공격 표면 증가

  3. 드라이버 버그가 kubelet 버그
     특정 스토리지 드라이버 버그 → kubelet 전체 영향

CSI 방식의 해결:
  스토리지 드라이버를 쿠버네티스 외부 프로세스로 분리
  표준 gRPC 인터페이스(CSI spec)로 kubelet과 통신
  드라이버를 쿠버네티스와 독립적으로 업데이트 가능
```

### CSI 드라이버 구조

CSI 드라이버는 두 개의 컴포넌트로 구성된다.

```
┌──────────────────────────────────────────────────────┐
│  Controller Plugin (Deployment 또는 StatefulSet)      │
│  → 클러스터당 1~몇 개 실행                             │
│                                                        │
│  담당 작업:                                            │
│    CreateVolume()           → 볼륨(디스크) 생성       │
│    DeleteVolume()           → 볼륨 삭제               │
│    ControllerPublishVolume() → 볼륨을 노드에 Attach   │
│    ControllerUnpublishVolume() → 노드에서 Detach      │
│    CreateSnapshot()         → 볼륨 스냅샷             │
│                                                        │
│  Sidecar들 (External Provisioner, Attacher 등):       │
│    쿠버네티스 API를 Watch해 Controller Plugin 호출    │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  Node Plugin (DaemonSet — 모든 노드에 실행)           │
│                                                        │
│  담당 작업:                                            │
│    NodeStageVolume()   → 노드에 볼륨 마운트 준비      │
│                           (파일시스템 포맷/마운트)     │
│    NodePublishVolume() → 파드 경로에 bind mount       │
│    NodeUnpublishVolume() → bind mount 해제            │
│    NodeUnstageVolume() → 노드 마운트 해제             │
│                                                        │
│  kubelet이 gRPC로 직접 호출                           │
│  Unix Socket: /var/lib/kubelet/plugins/<driver>/      │
└──────────────────────────────────────────────────────┘
```

### CSI Sidecar 컨테이너들

Controller Plugin 파드에는 표준 Sidecar들이 함께 배포된다. 이들이 쿠버네티스 API와 CSI 드라이버 사이의 다리 역할을 한다.

```
external-provisioner:
  PVC Watch → StorageClass provisioner 매칭 시 CreateVolume() 호출
  CreateVolume() 응답 받아 PV 오브젝트 생성

external-attacher:
  VolumeAttachment 오브젝트 Watch
  → ControllerPublishVolume() 호출 (노드에 디스크 Attach)
  → Attached: true 업데이트

external-snapshotter:
  VolumeSnapshot 오브젝트 Watch
  → CreateSnapshot() 호출

external-resizer:
  PVC 용량 변경 Watch
  → ControllerExpandVolume() 호출 (볼륨 확장)
```

### 볼륨 마운트 전체 흐름 (AWS EBS 예시)

```
파드 생성 → PVC 사용 설정됨
      │
      ▼
1. Scheduler가 파드를 worker-1에 배정

2. AD Controller (AttachDetach Controller in Controller Manager):
   파드가 필요한 볼륨을 worker-1에 Attach해야 함 감지
   → VolumeAttachment 오브젝트 생성:
     spec.attacher: ebs.csi.aws.com
     spec.nodeName: worker-1
     spec.source.persistentVolumeName: pv-xxx

3. external-attacher (CSI Controller Sidecar) Watch:
   VolumeAttachment 감지
   → CSI Controller Plugin: ControllerPublishVolume()
     (AWS API: ec2.AttachVolume(instance-id, vol-id))
   → VolumeAttachment.status.attached = true

4. worker-1의 kubelet:
   VolumeAttachment.attached = true 감지
   → CSI Node Plugin: NodeStageVolume()
     /dev/xvdf를 /var/lib/kubelet/plugins/kubernetes.io/csi/.../mount 에 마운트
     (파일시스템 포맷 또는 기존 fs 마운트)
   → CSI Node Plugin: NodePublishVolume()
     /var/lib/kubelet/pods/<uid>/volumes/.../ 에 bind mount

5. containerd에 컨테이너 실행 요청:
   config.json.mounts[] 에 파드 경로 포함
   → 컨테이너 내 /data에 bind mount

6. 파드 Running
```

### VolumeAttachment 오브젝트

```bash
# 볼륨이 노드에 Attach된 상태 확인
kubectl get volumeattachment

# NAME                                          ATTACHER           PV      NODE       ATTACHED
# csi-abc123def456...                           ebs.csi.aws.com    pv-xxx  worker-1   true

kubectl describe volumeattachment csi-abc123...
# Spec:
#   Attacher:  ebs.csi.aws.com
#   Node Name: worker-1
#   Source:
#     Persistent Volume Name: pv-xxx
# Status:
#   Attached: true        ← Attach 완료
#   Attachment Metadata:
#     devicePath: /dev/xvdf  ← 실제 디바이스 경로
```

---

## 💻 실전 실험

### 1. CSI 드라이버 구성 확인 (Kind)

```bash
# Kind는 local-path-provisioner 사용 (단순 CSI 유사 구현)
kubectl get pods -n local-path-storage
kubectl get storageclass

# CSIDriver 오브젝트 확인
kubectl get csidriver
# NAME                        ATTACHREQUIRED   PODINFOONMOUNT
# driver.longhorn.io          true             false
# (설치된 CSI 드라이버 목록)
```

### 2. VolumeAttachment 관찰

```bash
# PVC와 파드 생성
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-test-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: csi-test-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sleep', '3600']
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: csi-test-pvc
EOF

# VolumeAttachment 상태 확인
kubectl get volumeattachment -w
```

### 3. CSI Node Plugin 소켓 확인

```bash
docker exec -it kind-worker bash

# kubelet이 CSI Node Plugin과 통신하는 소켓
ls /var/lib/kubelet/plugins/

# 노드에서 마운트된 볼륨 확인
mount | grep /var/lib/kubelet/pods
```

### 4. 볼륨 마운트 단계 kubelet 로그 추적

```bash
# kubelet 로그에서 볼륨 마운트 과정
docker exec -it kind-control-plane bash
journalctl -u kubelet | grep -i "volume\|csi\|mount" | tail -30
```

### 5. CSI Controller Plugin 로그

```bash
# aws-ebs-csi-driver 예시 (EKS 클러스터)
kubectl logs -n kube-system -l app=ebs-csi-controller -c ebs-plugin | \
  grep -E "CreateVolume|AttachVolume" | tail -10
```

---

## 📊 CSI 연산별 담당 컴포넌트

| 연산 | 트리거 | 담당 컴포넌트 | CSI 호출 |
|-----|------|------------|--------|
| 볼륨 생성 | PVC 생성 | external-provisioner | CreateVolume() |
| 볼륨 삭제 | PV 삭제(Delete 정책) | external-provisioner | DeleteVolume() |
| 노드 Attach | 파드 스케줄링 | external-attacher | ControllerPublishVolume() |
| 노드 Detach | 파드 삭제 | external-attacher | ControllerUnpublishVolume() |
| 파드 마운트 | 파드 실행 | kubelet | NodeStageVolume() + NodePublishVolume() |
| 파드 언마운트 | 파드 종료 | kubelet | NodeUnpublishVolume() + NodeUnstageVolume() |
| 스냅샷 생성 | VolumeSnapshot 생성 | external-snapshotter | CreateSnapshot() |

---

## ⚖️ 트레이드오프

**Controller Plugin 배포 방식**

Controller Plugin을 Deployment로 배포하면 고가용성이 쉽지만, 동시에 여러 인스턴스가 같은 볼륨 연산을 실행할 수 있다. 대부분의 CSI 드라이버는 Leader Election을 사용해 하나의 Controller만 활성 상태로 유지한다. StatefulSet을 사용하면 순서 보장은 되지만 업그레이드 시 잠깐 중단이 발생한다.

**CSI와 스토리지 가용성**

CSI Controller Plugin이 다운되면 새로운 볼륨 생성/삭제/Attach가 불가능하다. 그러나 이미 마운트된 볼륨은 Node Plugin이 관리하므로 영향을 받지 않는다. 실행 중인 파드의 볼륨은 CSI Controller 장애와 무관하게 계속 동작한다.

---

## 📌 핵심 정리

```
CSI 등장 이유:
  in-tree 드라이버의 쿠버네티스 의존성 제거
  드라이버를 쿠버네티스와 독립적으로 업데이트 가능

Controller Plugin (Deployment):
  볼륨 생성/삭제, 노드 Attach/Detach
  external-provisioner/attacher Sidecar가 k8s API Watch

Node Plugin (DaemonSet):
  파일시스템 마운트, 파드 경로 bind mount
  kubelet이 gRPC로 직접 호출

볼륨 마운트 3단계:
  Provision → 디스크 생성 (Controller)
  Attach    → 노드에 연결 (Controller + VolumeAttachment)
  Mount     → 파드 경로에 마운트 (Node Plugin)

진단 도구:
  kubectl describe pvc → Provision 단계
  kubectl get volumeattachment → Attach 단계
  kubelet 로그, Node Plugin 로그 → Mount 단계
```

---

## 🤔 생각해볼 문제

**Q1.** CSI Node Plugin이 DaemonSet으로 배포되는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

Node Plugin은 각 노드에서 실제 파일시스템 마운트 연산을 수행해야 하기 때문이다. 마운트 작업은 해당 파드가 실행될 노드에서만 가능하다. kubelet이 CRI를 통해 컨테이너를 실행할 노드에서 Node Plugin에 직접 gRPC를 호출한다. 따라서 모든 노드에 Node Plugin이 있어야 하며, DaemonSet이 각 노드에 정확히 1개의 Node Plugin Pod를 보장한다.

</details>

**Q2.** 같은 EBS 볼륨을 두 파드가 동시에 마운트할 수 없는 이유는 CSI 레벨에서 어떻게 보장되는가?

<details>
<summary>해설 보기</summary>

두 가지 레벨에서 보장된다. 첫째, StorageClass에 `accessModes: ReadWriteOnce`가 설정되면 PVC-PV 바인딩 단계에서 하나의 PVC만 해당 PV에 바인딩될 수 있다. 둘째, AWS EBS 자체가 한 번에 하나의 EC2 인스턴스에만 Attach할 수 있다. `ControllerPublishVolume()`이 이미 다른 노드에 Attach된 볼륨에 대해 호출되면 AWS API가 오류를 반환한다. 이 두 레벨의 보호가 같은 볼륨의 동시 마운트를 방지한다.

</details>

**Q3.** 노드가 갑자기 죽었을 때, 해당 노드에 Attach된 볼륨은 어떻게 되는가?

<details>
<summary>해설 보기</summary>

노드가 죽으면 VolumeAttachment 오브젝트는 남아있지만 Attach 상태가 불명확해진다. 쿠버네티스는 안전을 위해 자동으로 Detach하지 않는다. 이를 Attach/Detach Controller의 `force-detach` 기능이 처리한다. 노드가 일정 시간(기본 6분) 동안 Not Ready 상태이면 Attach/Detach Controller가 강제 Detach를 수행하고 새 파드가 다른 노드에 스케줄링될 수 있다. AWS EBS는 반드시 한 인스턴스에서 완전히 Detach되어야 다른 인스턴스에 Attach할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: PV와 PVC — 동적 프로비저닝](./02-pv-pvc-storageclass.md)** | **[홈으로 🏠](../README.md)** | **[다음: StatefulSet — 순서 보장과 안정적 네트워크 ID ➡️](./04-statefulset.md)**

</div>
