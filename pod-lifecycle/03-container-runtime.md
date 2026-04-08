# 컨테이너 런타임 — containerd와 OCI 스펙

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- kubelet과 containerd는 어떤 인터페이스(CRI)로 통신하는가?
- containerd-shim은 왜 필요하고, 어떤 역할을 하는가?
- OCI Runtime Spec(config.json)은 무엇을 정의하는가?
- `crictl`로 kubelet을 거치지 않고 containerd를 직접 조회하는 방법은?
- docker-deep-dive에서 배운 namespace/cgroups가 여기서 어떻게 적용되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`kubectl describe pod`로 보이는 `ContainerCreating`에서 멈추는 장애는 kubelet과 containerd 사이에서 발생하는 경우가 많다. 이미지 Pull 실패, 볼륨 마운트 실패, 런타임 내부 오류 — 이 단계의 문제는 kubectl 레이어에서 보이지 않는다. `crictl`로 containerd를 직접 조회해야 원인이 보인다.

또한 containerd 없이 Docker를 Container Runtime으로 쓰던 시절부터 지금의 containerd + runc 구조까지 이해하면, `docker run`과 `kubectl run`이 내부에서 얼마나 비슷한 경로를 거치는지 명확해진다.

> 🔗 **docker-deep-dive 연결:** Linux namespace(PID/Network/Mount/UTS/IPC)와 cgroups(cpu.cfs_quota_us, memory.limit_in_bytes)의 원리는 docker-deep-dive에서 다뤘다. 여기서는 그 개념이 쿠버네티스 컨텍스트에서 어떻게 적용되는지 집중한다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 파드가 ContainerCreating에서 5분째 멈춤
      kubectl describe pod → Events에 오류 없음

원리를 모를 때의 판단:
  "kubectl이 이상하다" → 재시도
  "이미지 문제겠지" → 이미지 태그 확인 (정상)
  → 원인 못 찾음

실제 원인:
  containerd 내부에서 이미지 레이어 압축 해제 중 디스크 공간 부족
  kubectl 레이어에서는 "Pulling image..." 상태로만 보임
  
해결 방법:
  # containerd 직접 조회
  crictl ps -a | grep <파드 이름>  → 컨테이너 상태 확인
  crictl logs <container-id>       → 컨테이너 로그
  
  # 노드에서 containerd 로그
  journalctl -u containerd | tail -50
  
  # 디스크 사용량 확인
  df -h /var/lib/containerd
```

---

## ✨ 올바른 접근 (After — 런타임 계층을 이해한 진단)

```
ContainerCreating에서 멈춤 → 계층별 진단:

Layer 1. kubelet 레이어
  kubectl describe pod → Events 확인
  journalctl -u kubelet | grep <pod-name>

Layer 2. containerd 레이어
  crictl ps -a         → 컨테이너 상태 (Created/Running/Exited)
  crictl images        → 이미지 Pull 완료 여부
  crictl logs <id>     → 컨테이너 stdout/stderr

Layer 3. containerd 데몬 자체
  journalctl -u containerd | tail -100
  ls -lh /var/lib/containerd/io.containerd.content.v1.content/blobs/
  → 이미지 레이어 다운로드 상태

Layer 4. runc 레이어
  runc list            → 실행 중인 컨테이너 (OCI 레벨)
  /run/containerd/io.containerd.runtime.v2.task/ 확인
```

---

## 🔬 내부 동작 원리

### 전체 호출 체계

```
kubelet
  │
  │ gRPC (CRI: Container Runtime Interface)
  │ Unix Socket: /var/run/containerd/containerd.sock
  ▼
containerd (데몬)
  │ 이미지 관리 (스냅샷, 레이어, 메타데이터)
  │ 컨테이너 생명주기 관리
  │
  │ shim 생성 (컨테이너별 1개)
  ▼
containerd-shim-runc-v2
  │ containerd와 runc 사이의 중간 프로세스
  │ containerd 재시작 시에도 컨테이너 유지
  │
  │ OCI Runtime Spec (config.json) 전달
  ▼
runc
  │ Linux namespace 설정
  │ cgroups 설정
  │ 루트 파일시스템 마운트
  ▼
컨테이너 프로세스 (PID 1)
```

### CRI(Container Runtime Interface) 상세

CRI는 kubelet과 Container Runtime이 gRPC로 통신하는 인터페이스다. 두 개의 gRPC 서비스로 구성된다.

```
RuntimeService                ImageService
  RunPodSandbox()               PullImage()
  StopPodSandbox()              RemoveImage()
  RemovePodSandbox()            ImageStatus()
  CreateContainer()             ListImages()
  StartContainer()
  StopContainer()
  RemoveContainer()
  ListContainers()
  ContainerStatus()
  ExecSync()              ← kubectl exec 구현
  Attach()                ← kubectl attach 구현
  PortForward()           ← kubectl port-forward 구현
```

**PodSandbox — pause 컨테이너**

파드가 시작될 때 가장 먼저 `RunPodSandbox()`가 호출된다. 이 호출로 `pause` 컨테이너(infra 컨테이너)가 생성된다.

```
pause 컨테이너의 역할:
  - 파드의 Linux namespace들을 보유하고 초기화
    (Network, IPC, UTS namespace)
  - pause(2) 시스템콜로 시그널 대기만 하는 최소 프로세스
  - 메인 컨테이너들이 이 namespace에 join

파드 내 컨테이너가 네트워크를 공유하는 이유:
  모든 컨테이너가 pause 컨테이너의 Network namespace를 공유
  → 같은 IP, 같은 localhost, 같은 포트 공간 사용
  → 컨테이너끼리 localhost:8080으로 통신 가능
```

```bash
# pause 컨테이너 직접 확인
crictl ps -a | grep pause
# 파드마다 1개의 pause 컨테이너가 있음

# pause 컨테이너의 namespace 확인
PAUSE_PID=$(crictl inspect <pause-container-id> | jq '.info.pid')
ls /proc/$PAUSE_PID/ns/
# ipc  mnt  net  pid  uts  ...
```

### OCI(Open Container Initiative) Runtime Spec

runc가 컨테이너를 실행하기 위해 읽는 설정 파일(config.json)이다.

```json
{
  "ociVersion": "1.0.2",
  "process": {
    "user": {"uid": 0, "gid": 0},
    "args": ["/bin/nginx", "-g", "daemon off;"],
    "env": ["PATH=/usr/bin:/bin", "TERM=xterm"],
    "cwd": "/"
  },
  "root": {
    "path": "rootfs",    ← 컨테이너 루트 파일시스템 경로
    "readonly": false
  },
  "mounts": [
    {"destination": "/proc", "type": "proc", "source": "proc"},
    {"destination": "/dev",  "type": "devtmpfs", "source": "devtmpfs"},
    {
      "destination": "/etc/config",   ← ConfigMap 마운트
      "type": "bind",
      "source": "/var/lib/kubelet/pods/<uid>/volumes/configmap/config",
      "options": ["bind", "ro"]
    }
  ],
  "linux": {
    "namespaces": [
      {"type": "mount"},    ← Mount namespace (파일시스템 격리)
      {"type": "pid"},      ← PID namespace (프로세스 격리)
      {"type": "network", "path": "/proc/12345/ns/net"}  ← pause의 net ns 공유
    ],
    "resources": {
      "cpu": {
        "quota": 100000,    ← cpu.cfs_quota_us (100m CPU = 10000)
        "period": 100000    ← cpu.cfs_period_us
      },
      "memory": {
        "limit": 268435456  ← 256MiB (memory.limit_in_bytes)
      }
    },
    "cgroupsPath": "/kubepods/burstable/pod<uid>/<container-id>"
  }
}
```

이 파일이 `docker-deep-dive`에서 배운 namespace와 cgroups를 쿠버네티스 파드 스펙으로 변환한 실체다.

### containerd-shim의 역할

containerd와 runc 사이에 shim 프로세스가 있는 이유가 있다.

```
문제: containerd가 재시작되면 실행 중인 컨테이너가 죽는다
해결: containerd-shim이 runc와 컨테이너 사이에 위치

containerd 재시작 시나리오:
  runc → 컨테이너 시작 후 종료 (runc는 일회성)
  containerd-shim → 컨테이너 프로세스의 부모로 남아있음
  containerd 재시작 → shim을 통해 기존 컨테이너 재연결
  → 컨테이너는 중단 없이 계속 실행

shim의 추가 역할:
  컨테이너 stdout/stderr를 로그 파일로 수집
  컨테이너 exit code 추적
  OOM 이벤트 보고
```

### 이미지 레이어 관리

containerd는 이미지를 OCI Image Spec에 따라 레이어로 관리한다.

```
nginx 이미지 Pull 과정:
  1. Registry에서 Image Manifest 다운로드
     (레이어 목록과 각 레이어의 digest)

  2. 각 레이어를 Content Store에 다운로드
     /var/lib/containerd/io.containerd.content.v1.content/blobs/sha256/

  3. Snapshotter로 레이어 스택 구성
     각 레이어가 이전 레이어 위에 OverlayFS로 쌓임
     /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/

  4. 컨테이너 실행 시
     읽기 전용 레이어 스택 위에 쓰기 가능한 upper 레이어 추가
     → 컨테이너가 파일 수정해도 이미지 레이어는 변경 안 됨
```

---

## 💻 실전 실험

### 1. crictl로 containerd 직접 조회

```bash
# Kind 노드에 접속
docker exec -it kind-worker bash

# containerd 수준에서 실행 중인 모든 컨테이너
crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps

# pause 컨테이너 포함 전체 목록
crictl ps -a

# 특정 컨테이너 상세 정보 (JSON)
crictl inspect <container-id>

# 컨테이너 로그 (kubelet 없이 직접)
crictl logs <container-id>
crictl logs --tail=50 <container-id>

# 이미지 목록
crictl images
```

### 2. OCI config.json 직접 확인

```bash
# Kind 노드에 접속
docker exec -it kind-worker bash

# 실행 중인 컨테이너의 OCI spec 확인
CONTAINER_ID=$(crictl ps | grep nginx | awk '{print $1}' | head -1)
crictl inspect $CONTAINER_ID | jq '.info.runtimeSpec' | head -80

# runc 수준에서 확인 (runc list는 루트 권한 필요)
runc --root /run/containerd/runc/k8s.io list
```

### 3. cgroups 설정 확인

```bash
# nginx 파드의 cgroups 경로 확인
POD_UID=$(kubectl get pod nginx-test -o jsonpath='{.metadata.uid}')
ls /sys/fs/cgroup/memory/kubepods/besteffort/pod$POD_UID/

# 메모리 한도 확인 (0 = 무제한)
cat /sys/fs/cgroup/memory/kubepods/besteffort/pod$POD_UID/*/memory.limit_in_bytes

# Request 설정한 파드의 cgroups
kubectl run limited --image=nginx \
  --limits='memory=128Mi,cpu=200m'
kubectl get pod limited -o jsonpath='{.metadata.uid}'

# CPU quota 확인 (200m = 20000 / 100000)
cat /sys/fs/cgroup/cpu/kubepods/burstable/pod<uid>/*/cpu.cfs_quota_us
# 20000
```

### 4. pause 컨테이너 network namespace 공유 확인

```bash
# 파드 내 2개 컨테이너가 같은 network namespace를 공유하는 것 확인
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: shared-ns-test
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  - name: curl
    image: curlimages/curl
    command: ["sleep", "3600"]
EOF

# curl 컨테이너에서 nginx 컨테이너에 localhost로 접근
kubectl exec shared-ns-test -c curl -- curl localhost:80
# nginx 응답이 돌아옴 (같은 network namespace 공유)
```

### 5. 이미지 레이어 확인

```bash
docker exec -it kind-worker bash

# containerd의 이미지 스냅샷 확인
ls /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/

# OverlayFS 마운트 정보
cat /proc/mounts | grep overlay | head -5
# overlay /var/lib/containerd/.../rootfs overlay
#   lowerdir=<layer1>:<layer2>:<layer3>,
#   upperdir=<writable>,
#   workdir=<work>
```

---

## 📊 CRI 구현체 비교

| 항목 | containerd | CRI-O |
|-----|-----------|-------|
| 개발 주체 | CNCF (Docker 분리) | RedHat/IBM |
| 기본 채택 | Kubernetes 기본 (1.24~) | OpenShift |
| OCI 준수 | ✅ | ✅ |
| 이미지 지원 | OCI, Docker | OCI, Docker |
| 성능 | 높음 | 높음 |
| 주요 차이 | 범용적, 플러그인 생태계 풍부 | 쿠버네티스 전용으로 경량화 |
| shim | containerd-shim-runc-v2 | conmon |

---

## ⚖️ 트레이드오프

**CRI 표준화의 장점**

kubelet 코드 수정 없이 containerd, CRI-O 등 다양한 Runtime으로 교체 가능하다. GPU 특화 런타임(nvidia-container-runtime), 보안 강화 런타임(gVisor, Kata Containers)도 CRI를 구현하면 kubelet과 연동된다.

**containerd-shim의 오버헤드**

각 파드마다 shim 프로세스가 1개 생성된다. 노드에 100개 파드가 있으면 100개의 shim 프로세스가 실행된다. 대부분의 상황에서 무시할 수준이지만, 노드당 파드 수가 매우 많은 환경에서는 프로세스 수 제한(ulimit)을 확인해야 한다.

---

## 📌 핵심 정리

```
호출 체계:
  kubelet → (CRI gRPC) → containerd → (shim) → runc → 컨테이너 프로세스

CRI의 핵심:
  RuntimeService: 파드/컨테이너 생명주기 (RunPodSandbox, CreateContainer 등)
  ImageService: 이미지 관리 (PullImage, ListImages 등)
  kubectl exec/attach/port-forward도 CRI를 통해 구현

pause 컨테이너:
  파드의 network/IPC/UTS namespace 보유
  메인 컨테이너들이 pause의 network ns를 공유
  → 컨테이너끼리 localhost로 통신 가능

containerd-shim:
  containerd 재시작 시 컨테이너 유지
  컨테이너 로그 수집, exit code 추적

OCI config.json:
  namespace, cgroups, 마운트, 환경변수, 프로세스 정의
  파드 스펙 → kubelet → OCI spec으로 변환 → runc 실행
```

---

## 🤔 생각해볼 문제

**Q1.** `kubectl exec -it my-pod -- bash`는 내부적으로 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

kubectl → API Server에 ExecRequest 전송 → kubelet의 `/exec` 엔드포인트로 리다이렉트(SPDY 또는 WebSocket 업그레이드) → kubelet이 CRI의 `ExecSync()` 또는 `Exec()` 호출 → containerd가 이미 실행 중인 컨테이너의 network/mount namespace에 새 프로세스(`bash`)를 실행. 컨테이너 내부에서 bash가 실행되는 것은 기존 컨테이너 프로세스와 같은 namespace를 공유하는 새 프로세스다.

</details>

**Q2.** 같은 이미지를 사용하는 파드가 10개 있다. 노드에서 이미지 레이어는 몇 개 저장되는가?

<details>
<summary>해설 보기</summary>

이미지 레이어는 공유된다. containerd의 Content Store에 레이어가 digest 기준으로 한 번만 저장된다. 10개 파드 모두 같은 읽기 전용 레이어 스택(lowerdir)을 공유하고, 각 컨테이너별로 독립적인 쓰기 가능 레이어(upperdir)만 별도로 생성된다. 따라서 이미지가 캐시된 경우 추가적인 디스크 사용량은 컨테이너별 쓰기 레이어(보통 수십 MB 이하)뿐이다.

</details>

**Q3.** Kata Containers는 어떻게 CRI와 연동되어 더 강한 격리를 제공하는가?

<details>
<summary>해설 보기</summary>

Kata Containers는 CRI를 구현한다. kubelet의 관점에서는 일반 containerd와 동일한 CRI API를 사용한다. 차이는 내부 구현이다. 일반 containerd는 runc로 Linux namespace를 사용하는 반면, Kata Containers는 경량 VM(QEMU/KVM)을 생성해 그 안에서 컨테이너를 실행한다. 각 파드가 별도의 VM 커널 위에서 실행되므로 커널 취약점을 통한 탈출이 어려워진다. 대신 VM 시작 시간과 오버헤드가 추가된다.

</details>

---

> ⬅️ 이전: [02. 파드 스케줄링 알고리즘](./02-scheduling-algorithm.md)  
> ➡️ 다음: [04. 파드 종료 — SIGTERM과 graceful shutdown](./04-pod-termination.md)
