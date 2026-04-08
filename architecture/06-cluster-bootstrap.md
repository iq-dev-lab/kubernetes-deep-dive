# 클러스터 부트스트랩 — kubeadm과 Static Pod

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `kubeadm init`은 단계별로 어떤 작업을 수행하는가?
- Static Pod란 무엇이고, Control Plane 컴포넌트가 왜 Static Pod로 실행되는가?
- 닭이 먼저인가 달걀이 먼저인가 — kubelet 없이 API Server를 어떻게 시작하는가?
- 클러스터 인증서는 어떻게 생성되고 어디에 저장되는가?
- Control Plane이 스스로를 치유(self-healing)하는 원리는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Control Plane 컴포넌트가 갑자기 재시작되거나 인증서 만료 에러가 발생했을 때, 부트스트랩 과정을 이해하지 못하면 어디를 봐야 할지 알 수 없다. `/etc/kubernetes/manifests` 디렉토리, `/etc/kubernetes/pki` 인증서 경로, Static Pod의 동작 원리를 알아야 진단이 가능하다.

또한 kubeadm 없이 바이너리로 클러스터를 구성하는 "Kubernetes the Hard Way" 방식을 이해하고 싶다면, 부트스트랩 과정이 출발점이다. kubeadm은 이 과정을 자동화한 도구일 뿐이다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: kube-apiserver 파드가 갑자기 사라짐

원리를 모를 때의 판단:
  "파드가 없어졌으니 kubectl apply로 다시 만들어야겠다"
  → kubectl apply -f apiserver.yaml 시도
  → kubectl 자체가 API Server에 연결 못 해서 명령 실패
  → 패닉

실제 동작:
  kube-apiserver는 Static Pod
  /etc/kubernetes/manifests/kube-apiserver.yaml 파일이 있는 한
  kubelet이 자동으로 재시작시킴
  → 몇 초 기다리면 자동 복구

올바른 진단:
  kubectl get pod -n kube-system kube-apiserver-xxx → 없으면
  → kubectl logs -n kube-system kube-apiserver-xxx (마지막 로그)
  → cat /etc/kubernetes/manifests/kube-apiserver.yaml (스펙 확인)
  → journalctl -u kubelet | grep apiserver (kubelet 재시작 시도 로그)
```

---

## ✨ 올바른 접근 (After — 부트스트랩을 알고 난 진단)

```
Control Plane 컴포넌트 장애 시 진단 순서:

1. Static Pod 파일 확인
   ls /etc/kubernetes/manifests/
   → kube-apiserver.yaml이 있으면 kubelet이 재시작할 것

2. kubelet 상태 확인
   systemctl status kubelet
   journalctl -u kubelet | tail -50

3. 인증서 만료 여부 확인
   kubeadm certs check-expiration
   → 인증서 만료가 원인이면 kubeadm certs renew all

4. 컨테이너 런타임 상태 확인
   crictl ps -a | grep apiserver
   → 재시작 반복 중이면 컨테이너 로그 확인
   crictl logs <container-id>
```

---

## 🔬 내부 동작 원리

### 닭과 달걀 문제 — 어떻게 클러스터를 부트스트랩하는가

쿠버네티스 클러스터를 시작하는 데는 순환 의존성이 있다.

```
API Server를 시작하려면 → etcd가 있어야
etcd를 시작하려면     → 설정이 필요
Scheduler를 시작하려면 → API Server가 있어야
kubelet이 동작하려면  → API Server가 있어야

그렇다면 처음 시작은 어떻게?
```

해답이 **Static Pod**다. kubelet은 특별한 경우에 API Server 없이도 파드를 실행할 수 있다. `/etc/kubernetes/manifests/` 디렉토리의 YAML 파일을 직접 읽어 컨테이너를 실행한다.

```
부트스트랩 순서:

1. kubelet 프로세스 시작 (systemd 서비스)
   └── /etc/kubernetes/manifests/ 디렉토리 Watch 시작

2. 디렉토리에서 YAML 파일 발견
   ├── etcd.yaml           → etcd 컨테이너 시작
   ├── kube-apiserver.yaml → API Server 시작 (etcd 준비 후)
   ├── kube-scheduler.yaml → Scheduler 시작 (API Server 준비 후)
   └── kube-controller-manager.yaml → Controller Manager 시작

3. API Server가 올라온 후
   kubelet이 자기 자신을 API Server에 Node로 등록
   (일반 파드 관리 기능 활성화)

닭-달걀 해결:
  kubelet → Static Pod → etcd + API Server → 나머지 컴포넌트
  kubelet이 최초 부트스트랩의 핵심
```

### `kubeadm init` 단계별 동작

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```

```
Phase 1: preflight
  ├── 시스템 요구사항 확인 (CPU, 메모리, OS)
  ├── 필요한 바이너리 존재 여부 (kubelet, crictl)
  └── 포트 충돌 확인 (6443, 2379-2380, 10250-10252)

Phase 2: certs (인증서 생성)
  /etc/kubernetes/pki/ 에 생성:
  ├── ca.crt / ca.key                    (Cluster CA)
  ├── apiserver.crt / apiserver.key      (API Server 서버 인증서)
  ├── apiserver-kubelet-client.crt/key   (API Server → kubelet 인증)
  ├── front-proxy-ca.crt/key             (API Aggregation 레이어)
  ├── etcd/ca.crt / etcd/ca.key          (etcd CA)
  ├── etcd/server.crt / etcd/server.key  (etcd 서버 인증서)
  └── sa.pub / sa.key                    (ServiceAccount 토큰 서명)

Phase 3: kubeconfig
  /etc/kubernetes/ 에 생성:
  ├── admin.conf        (kubectl 관리자 설정)
  ├── kubelet.conf      (kubelet → API Server 인증)
  ├── controller-manager.conf
  └── scheduler.conf

Phase 4: control-plane (Static Pod 파일 생성)
  /etc/kubernetes/manifests/ 에 생성:
  ├── etcd.yaml
  ├── kube-apiserver.yaml
  ├── kube-controller-manager.yaml
  └── kube-scheduler.yaml
  → kubelet이 파일을 감지해 컨테이너 자동 시작

Phase 5: etcd
  Static Pod etcd 시작 완료 대기

Phase 6: wait-control-plane
  API Server 응답 대기 (최대 4분)
  /healthz 엔드포인트 폴링

Phase 7: upload-config
  kubeadm 설정을 ConfigMap으로 클러스터에 저장
  kubectl get configmap -n kube-system kubeadm-config

Phase 8: upload-certs (선택적)
  HA 구성 시 다른 control-plane 노드로 인증서 배포

Phase 9: mark-control-plane
  control-plane 노드에 Taint 추가
  node-role.kubernetes.io/control-plane:NoSchedule
  (일반 파드가 control-plane 노드에 스케줄링되지 않도록)

Phase 10: bootstrap-token
  워커 노드가 클러스터에 조인하기 위한 토큰 생성
  kubeadm join에 사용되는 토큰

Phase 11: kubelet-finalize
  kubelet의 클라이언트 인증서 갱신 설정

Phase 12: addon
  CoreDNS, kube-proxy 설치 (Deployment, DaemonSet으로)
```

### Static Pod의 동작 원리

Static Pod는 API Server 없이 kubelet이 직접 관리하는 파드다.

```
일반 파드:
  kubectl apply → API Server → etcd → Scheduler → kubelet → 컨테이너

Static Pod:
  /etc/kubernetes/manifests/file.yaml
          ↓
       kubelet (디렉토리 Watch)
          ↓
       containerd (컨테이너 실행)

특징:
  - API Server가 없어도 실행됨 (부트스트랩 가능)
  - 파일을 수정하면 kubelet이 자동으로 파드 재시작
  - 파일을 삭제하면 kubelet이 자동으로 파드 종료
  - kubectl로 삭제해도 파일이 남아있으면 재생성됨 (자가 치유)
  - API Server에는 Mirror Pod로 표시됨 (읽기 전용)
```

```bash
# Static Pod 파일 수정 → 즉시 반영
# (예: kube-apiserver의 기능 플래그 추가)
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# 저장하면 kubelet이 자동으로 새 설정으로 재시작

# API Server에 Mirror Pod 확인
kubectl get pod -n kube-system kube-apiserver-control-plane
# 삭제를 시도해도 파일이 있으면 재생성됨
kubectl delete pod -n kube-system kube-apiserver-control-plane
# pod "kube-apiserver-control-plane" deleted
# (곧바로 재생성됨)
```

### 인증서 구조와 만료 관리

쿠버네티스의 모든 컴포넌트 간 통신은 mTLS로 암호화된다.

```
인증서 계층:
  Cluster CA (10년 유효)
    ├── API Server 서버 인증서 (1년)
    ├── API Server 클라이언트 인증서 (1년)
    ├── Controller Manager 클라이언트 인증서 (1년)
    ├── Scheduler 클라이언트 인증서 (1년)
    └── kubelet 클라이언트 인증서 (1년, 자동 갱신)

  etcd CA (10년 유효)
    ├── etcd 서버 인증서 (1년)
    └── etcd 피어 인증서 (1년)
```

```bash
# 인증서 만료일 확인
kubeadm certs check-expiration

# 출력:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME
# admin.conf                 Jan 01, 2025 00:00 UTC   364d
# apiserver                  Jan 01, 2025 00:00 UTC   364d
# ...

# 인증서 갱신
kubeadm certs renew all

# kubeconfig 갱신 후 복사 (kubectl 사용자용)
cp /etc/kubernetes/admin.conf ~/.kube/config
```

kubelet의 클라이언트 인증서는 `certificate-controller`(Controller Manager 내)가 자동으로 갱신한다. 단, CA 인증서 자체는 자동 갱신되지 않으므로 10년마다 수동 작업이 필요하다.

### 워커 노드 조인 과정

```bash
# control-plane에서 조인 명령어 생성
kubeadm token create --print-join-command

# 워커 노드에서 실행
kubeadm join <api-server-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

```
조인 과정:
1. 토큰으로 API Server에 임시 인증
2. CA 인증서 해시로 API Server 진위 확인 (MITM 방지)
3. kubelet 클라이언트 인증서 요청 (CSR)
4. Controller Manager가 CSR 승인 → 인증서 발급
5. kubelet이 인증서로 API Server에 Node 등록
6. /etc/kubernetes/manifests/ 디렉토리 Watch 시작
7. kube-proxy DaemonSet에 의해 kube-proxy 파드 자동 시작
```

---

## 💻 실전 실험

### 1. Kind 클러스터에서 Static Pod 확인

```bash
# Kind의 control-plane 노드에 접속
docker exec -it kind-control-plane bash

# Static Pod 파일 목록
ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml

# API Server Static Pod 스펙 확인
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -A5 "command:"
```

### 2. 인증서 디렉토리 구조 확인

```bash
docker exec -it kind-control-plane bash

# 인증서 디렉토리
ls /etc/kubernetes/pki/
# apiserver.crt  apiserver.key  ca.crt  ca.key  ...

# 인증서 내용 확인 (만료일, Subject 등)
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | \
  grep -E "Subject:|Not After"
```

### 3. kubelet이 Static Pod를 재시작하는 과정 관찰

```bash
# Kind 노드에서
docker exec -it kind-control-plane bash

# kube-scheduler Static Pod 파일 임시 이동 (Scheduler 중단)
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/

# kubelet이 Scheduler 컨테이너를 종료함
# kubectl get pod -n kube-system | grep scheduler → 없어짐
# 새 파드 생성하면 Pending 상태 유지

# 파일 복원 → kubelet이 자동으로 Scheduler 재시작
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
# kubectl get pod -n kube-system | grep scheduler → 다시 Running
```

### 4. kubeconfig 인증 정보 분석

```bash
# admin.conf의 사용자 인증서 분석
cat ~/.kube/config | grep client-certificate-data | \
  awk '{print $2}' | base64 -d | \
  openssl x509 -text -noout | grep -E "Subject:|Not After"

# 출력:
# Subject: O=system:masters, CN=kubernetes-admin
# Not After : Jan 01, 2025 00:00:00 UTC
```

### 5. 전체 클러스터 인증서 만료일 한번에 확인

```bash
# kubeadm certs는 control-plane 노드에서만 동작
docker exec -it kind-control-plane \
  kubeadm certs check-expiration 2>/dev/null
```

---

## 📊 Static Pod vs 일반 Pod 비교

| 항목 | Static Pod | 일반 Pod |
|-----|-----------|---------|
| 관리 주체 | kubelet (파일 기반) | API Server + 컨트롤러 |
| API Server 의존성 | 없음 (부트스트랩 가능) | 필요 |
| 파일 삭제 시 | 컨테이너 종료 | 무관 |
| kubectl delete | 재생성됨 (파일이 남아있으면) | 영구 삭제 |
| Scheduler 배치 | 해당 없음 (해당 노드 고정) | Scheduler가 결정 |
| 사용 사례 | Control Plane 컴포넌트 | 일반 워크로드 |

---

## ⚖️ 트레이드오프

**Static Pod의 장점**
- API Server 없이 실행 가능 → 클러스터 부트스트랩의 핵심
- 파일 수정만으로 컴포넌트 재설정 가능
- 자가 치유: kubelet이 항상 재시작 보장

**Static Pod의 단점**
- 특정 노드에 고정됨 → Scheduler의 최적화 혜택 없음
- 멀티 노드 HA 구성 시 각 노드마다 수동 설정 필요
- DaemonSet처럼 전체 노드 배포 자동화 불가

**kubeadm의 한계**
- 기본 구성은 단일 control-plane (HA 구성은 추가 작업 필요)
- 클라우드별 특화 기능(AWS LB, GKE 오토스케일링)은 별도 설치 필요
- 대규모 클러스터 설정 커스터마이징이 제한적 → kops, Cluster API 등 사용

---

## 📌 핵심 정리

```
부트스트랩 순환 의존 해결법 = Static Pod
  kubelet이 /etc/kubernetes/manifests/ 파일을 직접 읽어 실행
  API Server 없이도 etcd, API Server 자체를 시작 가능

kubeadm init 핵심 단계:
  1. 인증서 생성 (/etc/kubernetes/pki/)
  2. kubeconfig 생성 (/etc/kubernetes/*.conf)
  3. Static Pod 파일 생성 (/etc/kubernetes/manifests/)
  4. kubelet이 파일 감지 → Control Plane 컨테이너 시작

Control Plane 자가 치유:
  Static Pod 파일이 있는 한 kubelet이 항상 재시작
  kubectl delete로 삭제해도 재생성됨
  파일을 삭제해야 진짜 종료

인증서 관리:
  일반 컴포넌트 인증서: 1년 유효, kubeadm certs renew로 갱신
  kubelet 클라이언트 인증서: Controller Manager가 자동 갱신
  CA 인증서: 10년, 수동 갱신 필요
```

---

## 🤔 생각해볼 문제

**Q1.** control-plane 노드의 `/etc/kubernetes/manifests/kube-apiserver.yaml` 파일을 잘못 수정해 API Server가 시작되지 않는다. `kubectl`을 사용할 수 없는 상황에서 어떻게 복구하는가?

<details>
<summary>해설 보기</summary>

kubelet이 직접 Static Pod 파일을 읽으므로, 파일을 수정하면 된다. (1) control-plane 노드에 SSH로 접속. (2) `vi /etc/kubernetes/manifests/kube-apiserver.yaml`로 잘못된 설정 수정. (3) 파일을 저장하면 kubelet이 자동으로 새 설정으로 API Server 재시작. (4) `crictl ps | grep apiserver`로 컨테이너가 올라왔는지 확인. kubectl은 API Server가 복구된 후에 사용 가능하다.

</details>

**Q2.** HA control-plane 구성에서 3개의 control-plane 노드 각각에 API Server가 실행된다. 클라이언트(kubectl)는 어느 API Server에 연결하는가?

<details>
<summary>해설 보기</summary>

HA 구성에서는 API Server들 앞에 로드밸런서를 둔다. kubeconfig의 `server` 필드가 이 로드밸런서 주소를 가리킨다. 요청은 로드밸런서가 살아있는 API Server 중 하나로 분배한다. 하나의 API Server가 죽어도 로드밸런서가 나머지로 라우팅하므로 클라이언트는 영향을 받지 않는다. etcd는 Raft로 일관성을 보장하므로, 어느 API Server에 쓰더라도 동일한 etcd에 저장된다.

</details>

**Q3.** `kubeadm join` 과정에서 `--discovery-token-ca-cert-hash`가 필요한 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

중간자 공격(MITM) 방지를 위해서다. 워커 노드가 "이 API Server가 진짜 우리 클러스터의 API Server인가?"를 검증해야 한다. `--token`만으로는 가짜 API Server에 연결될 수 있다. `--discovery-token-ca-cert-hash`는 CA 인증서의 SHA256 해시로, 연결된 API Server의 CA 인증서 해시와 비교한다. 일치하면 진짜 클러스터임이 검증된다. 이후 kubelet이 이 CA를 신뢰하는 클라이언트 인증서를 발급받아 정식 멤버가 된다.

</details>

---

<div align="center">

**[⬅️ 이전: kubelet — 노드 에이전트와 Probe](./05-kubelet-probes.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — 파드 생성 전 과정 ➡️](../pod-lifecycle/01-pod-creation-full-flow.md)**

</div>
