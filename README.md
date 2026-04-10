<div align="center">

# ☸️ Kubernetes Deep Dive

**"kubectl apply로 파드를 배포하는 것과, Control Plane이 수천 개의 파드 상태를 어떻게 선언적으로 수렴시키는지 아는 것은 다르다"**

<br/>

> *"`kubectl apply`를 쓰지만 API Server가 내부에서 무슨 API 호출을 하는지 모른다 — 와 — etcd의 원하는 상태와 현재 상태를 Reconciliation Loop가 어떻게 일치시키는지, kube-proxy가 ClusterIP를 iptables 레벨에서 어떻게 라우팅하는지 아는 것의 차이를 만드는 레포"*

단일 `kubectl apply`가 API Server → etcd → Scheduler → kubelet → containerd → runc까지 어떻게 흘러가는지,  
ReplicaSet Controller가 파드 수를 유지하기 위해 etcd를 어떻게 Watch하는지, ClusterIP가 실제 IP 없이 iptables DNAT로 동작하는 원리까지  
**왜 이렇게 설계됐는가** 라는 질문으로 쿠버네티스 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io/docs/)
[![Kind](https://img.shields.io/badge/Kind-로컬_클러스터-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kind.sigs.k8s.io/)
[![Docs](https://img.shields.io/badge/Docs-38개-blue?style=flat-square&logo=readthedocs&logoColor=white)](https://github.com/dev-book-lab/kubernetes-deep-dive)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

쿠버네티스에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`kubectl apply`하면 파드가 생성됩니다" | `kubectl apply` → API Server(인증/인가/Admission) → etcd 저장 → Scheduler Watch → kubelet Watch → containerd → runc까지 각 단계의 실제 API 호출과 이벤트 흐름 |
| "Service를 만들면 로드밸런싱됩니다" | ClusterIP가 실제 IP 없이 동작하는 이유, kube-proxy가 `KUBE-SERVICES → KUBE-SVC-<hash> → KUBE-SEP-<hash>`로 DNAT 규칙을 생성하는 방식, `iptables-save`로 직접 확인 |
| "etcd가 클러스터 상태를 저장합니다" | Raft Leader Election과 Log Replication이 어떻게 일관성을 보장하는지, Quorum이 가용성과 일관성을 동시에 결정하는 방식, `etcdctl get /registry/pods` 원본 데이터 확인 |
| "ReplicaSet이 파드 수를 유지합니다" | Controller Manager의 Reconciliation Loop가 Desired(3) vs Actual(2)를 비교해 새 Pod 오브젝트를 생성 요청하는 정확한 메커니즘 — "선언형"이 내부에서 의미하는 것 |
| "Liveness Probe가 컨테이너를 재시작합니다" | kubelet이 HTTP/TCP/exec 방식으로 Probe를 실행하는 주기, 실패 임계값 계산, `kubectl describe pod`의 Events 필드에서 실패 이력 추적 |
| "HPA가 파드를 스케일아웃합니다" | Metrics Server가 kubelet Summary API로 CPU/메모리를 수집하는 경로, HPA 컨트롤러가 `desiredReplicas = ceil(currentReplicas * currentMetric / desiredMetric)`으로 계산하는 알고리즘 |
| 이론 나열 | Kind 기반 로컬 3노드 클러스터 실험 + `kubectl describe`, `kubectl get events`, `iptables-save`, `etcdctl get` 결과를 직접 분석 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Ch1](https://img.shields.io/badge/🔹_Ch1-클러스터_아키텍처_개요-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](./architecture/01-cluster-architecture-overview.md)
[![Ch2](https://img.shields.io/badge/🔹_Ch2-파드_생성_전_과정-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](./pod-lifecycle/01-pod-creation-full-flow.md)
[![Ch3](https://img.shields.io/badge/🔹_Ch3-파드_네트워킹_기초-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](./networking/01-pod-networking-basics.md)
[![Ch4](https://img.shields.io/badge/🔹_Ch4-Volume과_파드-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](./storage/01-volumes.md)
[![Ch5](https://img.shields.io/badge/🔹_Ch5-Request와_Limit-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](./resource-scaling/01-request-limit-cgroups.md)
[![Ch6](https://img.shields.io/badge/🔹_Ch6-롤링_업데이트-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](./deployment-operations/01-rolling-update.md)
[![Ch7](https://img.shields.io/badge/🔹_Ch7-Operator_패턴-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](./advanced-patterns/01-operator-crd.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Kubernetes 아키텍처 — Control Plane과 Data Plane

> **핵심 질문:** Control Plane의 각 컴포넌트는 어떻게 서로 통신하는가? API Server는 요청을 어떻게 처리하고, etcd는 어떻게 일관성을 보장하며, kubelet은 어떻게 컨테이너를 실행하는가?

<details>
<summary><b>전체 아키텍처 개요부터 클러스터 부트스트랩까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 클러스터 아키텍처 개요 — Control Plane vs Data Plane](./architecture/01-cluster-architecture-overview.md) | API Server / etcd / Scheduler / Controller Manager로 구성된 Control Plane과 kubelet / kube-proxy / Container Runtime으로 구성된 Data Plane의 역할 분리, 각 컴포넌트가 API Server를 통해서만 통신하는 이유, Watch 메커니즘으로 상태 변화를 감지하는 방식 |
| [02. API Server — 모든 요청의 진입점](./architecture/02-api-server.md) | 인증(Bearer Token / Client Certificate) → 인가(RBAC) → Admission Controller(Mutating → Validating) 파이프라인의 각 단계, Watch API가 long-polling으로 컴포넌트들에게 변경 이벤트를 전달하는 방식, `kubectl api-resources`와 `--v=8` 플래그로 실제 API 호출 추적 |
| [03. etcd와 Raft 합의 알고리즘](./architecture/03-etcd-raft.md) | etcd가 클러스터 상태의 단일 진실 공급원인 이유, Raft Leader Election(term/vote)과 Log Replication이 노드 장애 시 일관성을 보장하는 방식, Quorum(N/2+1)이 가용성과 일관성을 동시에 결정하는 공식, `etcdctl get /registry/pods` 원본 데이터 직접 확인 |
| [04. Controller Manager — Reconciliation Loop의 실체](./architecture/04-controller-manager-reconciliation.md) | Desired State vs Actual State 비교가 실제 코드에서 어떻게 구현되는지, ReplicaSet Controller가 파드 수를 유지하기 위해 etcd를 Watch하고 Pod 오브젝트를 생성 요청하는 정확한 메커니즘, "명령형"이 아닌 "선언형"이 내부에서 의미하는 것 |
| [05. kubelet — 노드 에이전트와 Probe](./architecture/05-kubelet-probes.md) | kubelet이 API Server를 Watch해 자신 노드에 배정된 Pod 스펙을 감지하고 CRI(Container Runtime Interface)로 컨테이너 실행을 요청하는 흐름, Liveness / Readiness / Startup Probe가 HTTP / TCP / exec 방식으로 컨테이너 상태를 판단하는 구체적 메커니즘, `kubectl describe pod`의 Events로 Probe 실패 이력 추적 |
| [06. 클러스터 부트스트랩 — kubeadm과 Static Pod](./architecture/06-cluster-bootstrap.md) | `kubeadm init`이 CA 인증서를 생성하고 etcd / API Server / Scheduler / Controller Manager를 Static Pod으로 실행하는 전 과정, kubelet이 `/etc/kubernetes/manifests`를 Watch해 Static Pod를 관리하는 방식, Control Plane이 스스로를 치유(self-healing)하는 원리 |

</details>

<br/>

### 🔹 Chapter 2: 파드의 생명주기 — 생성부터 종료까지

> **핵심 질문:** `kubectl apply`를 실행한 순간부터 컨테이너가 Running 상태가 되기까지 어떤 컴포넌트가 어떤 순서로 개입하는가? 파드가 종료될 때 연결이 끊기지 않으려면 무엇이 필요한가?

<details>
<summary><b>파드 생성 전 과정부터 Init Container까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 파드 생성 전 과정 — kubectl부터 컨테이너 시작까지](./pod-lifecycle/01-pod-creation-full-flow.md) | `kubectl apply` → API Server(인증/인가/Admission) → etcd 저장(nodeName="") → Scheduler Watch → Filtering/Scoring → Bind(nodeName="worker-1") → kubelet Watch → 이미지 Pull → CNI → containerd → runc → 상태 업데이트까지 각 단계의 실제 동작과 `kubectl get events`로 타임라인 추적 |
| [02. 파드 스케줄링 알고리즘 — Filtering과 Scoring](./pod-lifecycle/02-scheduling-algorithm.md) | Filtering(Predicates) 단계에서 부적합 노드를 제거하는 기준(리소스 부족 / Taint / NodeAffinity / PodAntiAffinity), Scoring(Priorities) 단계에서 LeastAllocated / InterPodAffinity 등으로 점수를 계산해 최적 노드를 선택하는 알고리즘, `kubectl describe pod`의 Events에서 스케줄링 결정 추적 |
| [03. 컨테이너 런타임 — containerd와 OCI 스펙](./pod-lifecycle/03-container-runtime.md) | kubelet → CRI gRPC → containerd → containerd-shim → runc 호출 체계, OCI Runtime Spec(config.json)이 컨테이너의 namespace / cgroups / mount를 정의하는 방식, `crictl ps`와 `crictl inspect`로 containerd 레벨에서 컨테이너 상태 확인 |
| [04. 파드 종료 — SIGTERM과 graceful shutdown](./pod-lifecycle/04-pod-termination.md) | `kubectl delete pod` → DeletionTimestamp 설정 → preStop Hook 실행 → SIGTERM 전달 → terminationGracePeriodSeconds 대기 → SIGKILL 순서, preStop Hook에서 `sleep`으로 iptables 전파 지연을 기다려야 in-flight 요청이 끊기지 않는 이유, `terminationGracePeriodSeconds`가 preStop 실행 시간을 포함해 계산된다는 주의사항 |
| [05. Init Container와 Sidecar 패턴](./pod-lifecycle/05-init-container-sidecar.md) | Init Container가 순서대로 실행 완료되어야 메인 컨테이너가 시작되는 보장 메커니즘, Sidecar(Envoy Proxy / Fluentd)가 파드 네트워크 네임스페이스를 공유해 트래픽을 가로채는 방식, Kubernetes 1.29에서 정식 도입된 Sidecar Container 개념과 기존 방식과의 차이 |

</details>

<br/>

### 🔹 Chapter 3: Kubernetes 네트워킹 완전 분해

> **핵심 질문:** 파드 IP는 어떻게 할당되고, ClusterIP는 실제로 존재하지 않는데 어떻게 동작하며, Ingress Controller는 L7 라우팅을 어떻게 구현하는가?

<details>
<summary><b>파드 네트워킹 기초부터 Network Policy까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 파드 네트워킹 기초 — IP-per-Pod 모델](./networking/01-pod-networking-basics.md) | 모든 파드가 고유한 IP를 갖는 쿠버네티스 네트워킹 모델, 파드 간 통신이 NAT 없이 이루어지는 원리, 파드 내 컨테이너들이 네트워크 네임스페이스를 공유하는 방식, `pause(infra)` 컨테이너가 네트워크 네임스페이스를 초기화하고 유지하는 역할 |
| [02. CNI — Flannel과 Calico 내부 동작](./networking/02-cni-plugins.md) | kubelet이 CNI 플러그인을 호출해 파드 IP를 할당하는 방식, Flannel의 VXLAN 오버레이(UDP 캡슐화로 노드 간 패킷 전달)와 Calico의 BGP 라우팅(언더레이 네트워크 직접 활용)의 내부 동작 차이, `ip route`와 `bridge fdb`로 라우팅 테이블 직접 확인 |
| [03. Service와 kube-proxy — iptables 규칙 생성](./networking/03-service-kube-proxy.md) | ClusterIP가 실제 IP 없이 동작하는 이유(어떤 인터페이스에도 바인딩되지 않음), kube-proxy가 `PREROUTING → KUBE-SERVICES → KUBE-SVC-<hash> → KUBE-SEP-<hash>` 체인으로 DNAT 규칙을 생성하는 방식, `iptables-save | grep <service>`로 실제 규칙 확인, IPVS 모드의 차이 |
| [04. Service 종류 완전 분해](./networking/04-service-types.md) | ClusterIP(클러스터 내부 전용) / NodePort(모든 노드의 특정 포트로 외부 접근) / LoadBalancer(클라우드 LB 프로비저닝) / ExternalName(CNAME 리다이렉션) 각각의 구현 방식, NodePort가 모든 노드에서 동일하게 동작하는 이유(kube-proxy가 모든 노드에서 동일한 iptables 규칙을 생성) |
| [05. Ingress — L7 라우팅과 Nginx Controller](./networking/05-ingress.md) | Ingress Controller(Nginx)가 Ingress 리소스를 Watch해 `nginx.conf`를 자동 생성하는 방식, 호스트 기반 라우팅과 경로 기반 라우팅이 Nginx 설정에서 어떻게 표현되는지, TLS 종료(Termination)와 백엔드 Service 연결, `kubectl logs -n ingress-nginx`로 요청 라우팅 추적 |
| [06. CoreDNS — Service 이름 해석](./networking/06-coredns.md) | CoreDNS가 `<service>.<namespace>.svc.cluster.local` 형식의 쿼리를 ClusterIP로 변환하는 방식, `/etc/resolv.conf`의 `ndots:5` 설정이 DNS 조회 횟수에 미치는 영향(불필요한 FQDN 탐색), `kubectl exec`으로 파드 내 DNS 조회 과정 추적 |
| [07. Network Policy — 파드 간 트래픽 제어](./networking/07-network-policy.md) | Network Policy가 iptables / eBPF 레벨에서 파드 간 트래픽을 필터링하는 원리, Default Deny 정책이 동작하는 방식(Network Policy가 없으면 모든 트래픽 허용이 기본), Calico / Cilium이 Network Policy를 구현하는 방식의 차이, `kubectl exec`으로 정책 적용 전후 트래픽 비교 |

</details>

<br/>

### 🔹 Chapter 4: 스토리지 — 데이터 영속성의 원리

> **핵심 질문:** 파드가 재시작되어도 데이터가 유지되려면 무엇이 필요한가? CSI는 어떻게 스토리지 드라이버를 플러그인 방식으로 교체 가능하게 만드는가?

<details>
<summary><b>Volume 마운트 원리부터 스토리지 성능 고려사항까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Volume과 파드 — emptyDir부터 configMap까지](./storage/01-volumes.md) | emptyDir(파드 수명과 동일한 임시 스토리지, tmpfs 옵션으로 메모리에 저장) / hostPath(노드 파일시스템 직접 마운트, 보안 위험) / configMap / secret이 각각 kubelet에 의해 어떻게 마운트되는지, 컨테이너 재시작 vs 파드 재시작 시 데이터 유지 여부 차이 |
| [02. PV와 PVC — 동적 프로비저닝](./storage/02-pv-pvc-storageclass.md) | PVC가 StorageClass를 통해 클라우드 디스크를 자동 생성(동적 프로비저닝)하는 과정, PV-PVC Binding 알고리즘(용량 / AccessMode / StorageClass 매칭), PV의 Reclaim Policy(Retain / Delete)가 PVC 삭제 시 디스크에 미치는 영향(`Retain`은 PVC 삭제 후에도 디스크 보존, `Delete`는 즉시 삭제), `kubectl get pv`로 바인딩 상태 확인 |
| [03. CSI — 스토리지 드라이버 표준 인터페이스](./storage/03-csi.md) | CSI(Container Storage Interface)가 등장하기 전 in-tree 플러그인의 문제점(커널 릴리즈 의존성), CSI Driver가 Controller Plugin(볼륨 생성/삭제)과 Node Plugin(Attach/Mount)으로 분리되는 구조, kubelet이 CSI Node Plugin을 호출해 볼륨을 노드에 마운트하는 단계별 과정 |
| [04. StatefulSet — 순서 보장과 안정적 네트워크 ID](./storage/04-statefulset.md) | StatefulSet이 파드를 순서대로 생성/삭제하는 보장 메커니즘(`pod-0` → `pod-1` → `pod-2`), Headless Service로 `pod-0.<service>.<namespace>.svc.cluster.local` 형식의 안정적 DNS를 제공하는 방식, `volumeClaimTemplates`로 각 파드가 고유한 PVC를 갖는 원리, 파드 재생성 후 동일 PVC에 재연결되는 과정 |
| [05. 스토리지 성능 고려사항](./storage/05-storage-performance.md) | ReadWriteOnce(단일 노드 읽기/쓰기) vs ReadWriteMany(다중 노드 읽기/쓰기) AccessMode의 실제 제약, 로컬 PV(노드 디스크 직접 사용)와 원격 스토리지(NFS / EBS)의 latency 차이, 데이터베이스를 쿠버네티스에서 운영할 때의 트레이드오프(StatefulSet + 로컬 PV vs 관리형 RDS) |

</details>

<br/>

### 🔹 Chapter 5: 자원 관리와 스케일링

> **핵심 질문:** Request와 Limit의 차이는 무엇이고 cgroups로 어떻게 구현되는가? HPA는 어떤 메트릭 파이프라인을 통해 파드 수를 계산하는가?

<details>
<summary><b>cgroups 기반 Request/Limit부터 클러스터 오토스케일러까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Request와 Limit — cgroups 구현](./resource-scaling/01-request-limit-cgroups.md) | Request가 스케줄링 기준(Scheduler가 노드 잔여 Request 합산으로 배치 결정)이고 Limit이 실제 사용 상한(cgroups cpu.cfs_quota_us / memory.limit_in_bytes로 구현)인 이유, Limit 없는 파드가 노드 전체에 미치는 영향, `cat /sys/fs/cgroup/memory/...`로 Limit 적용 확인 |
| [02. QoS 클래스 — OOM 발생 시 파드 종료 순서](./resource-scaling/02-qos-classes.md) | Guaranteed(Request == Limit) / Burstable(Request < Limit) / BestEffort(Request/Limit 미설정) 분류 기준, OOM Killer가 `oom_score_adj` 값을 기준으로 BestEffort → Burstable → Guaranteed 순으로 파드를 종료하는 원리, `kubectl describe node`의 `Conditions`에서 메모리 압박 상태 확인 |
| [03. HPA — 메트릭 파이프라인과 스케일 알고리즘](./resource-scaling/03-hpa.md) | kubelet Summary API → Metrics Server → API Server metrics.k8s.io → HPA Controller 메트릭 수집 경로, `desiredReplicas = ceil(currentReplicas × currentMetric / desiredMetric)` 알고리즘, 스케일다운 쿨다운 기간(기본 5분)이 필요한 이유, 커스텀 메트릭(Prometheus Adapter)으로 확장하는 방식 |
| [04. VPA — Request/Limit 자동 조정](./resource-scaling/04-vpa.md) | VPA Recommender가 과거 사용량을 분석해 Request/Limit 권고값을 계산하는 방식, VPA Admission Controller(Mutating Webhook)가 파드 생성 시 스펙을 자동 수정하는 원리, HPA와 VPA를 동시에 사용할 때 충돌이 발생하는 이유와 회피 방법 |
| [05. 클러스터 오토스케일러 — 노드 추가와 스케일다운](./resource-scaling/05-cluster-autoscaler.md) | Pending 파드 감지(스케줄링 실패 이유가 Insufficient resource인 경우) → 노드 그룹 시뮬레이션 → 클라우드 API(AWS ASG / GKE MIG) 호출로 노드 추가하는 과정, 스케일다운 시 파드를 다른 노드로 이동(Drain)하고 노드를 삭제하는 원리, PodDisruptionBudget으로 스케일다운 중 최소 가용성을 보장하는 방법 |

</details>

<br/>

### 🔹 Chapter 6: 배포 전략과 운영

> **핵심 질문:** 롤링 업데이트 중 maxSurge/maxUnavailable은 어떻게 파드 교체 속도를 제어하는가? 무중단 배포를 위해 Readiness Probe, preStop Hook, terminationGracePeriodSeconds가 삼각편대를 이루는 이유는?

<details>
<summary><b>롤링 업데이트 알고리즘부터 모니터링 파이프라인까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Deployment 롤링 업데이트 — ReplicaSet 교체 알고리즘](./deployment-operations/01-rolling-update.md) | `maxSurge`(최대 초과 파드 수)와 `maxUnavailable`(최대 불가용 파드 수) 설정이 새 ReplicaSet 생성 속도와 기존 ReplicaSet 축소 속도를 제어하는 방식, 롤백이 이전 ReplicaSet을 다시 스케일업하는 방식(`kubectl rollout undo`), `kubectl rollout history`로 ReplicaSet 리비전 확인 |
| [02. 무중단 배포 — Probe와 Hook의 삼각편대](./deployment-operations/02-zero-downtime-deploy.md) | 파드 종료 시 Readiness Probe 실패 → 엔드포인트 제거 → SIGTERM 전달 순서가 보장되지 않으면 in-flight 요청이 끊기는 이유, preStop Hook에서 `sleep`으로 엔드포인트 제거 전파를 기다리는 패턴, `terminationGracePeriodSeconds`가 preStop Hook 실행 시간을 포함한다는 주의사항 |
| [03. RBAC — 최소 권한 원칙 구현](./deployment-operations/03-rbac.md) | Role(네임스페이스 범위) / ClusterRole(클러스터 범위) / RoleBinding / ClusterRoleBinding 계층 구조, ServiceAccount가 Pod에 마운트된 토큰으로 API Server에 인증하는 방식, `kubectl auth can-i`로 권한 확인, 최소 권한 원칙 적용 시 흔히 빠지는 함정 |
| [04. ConfigMap과 Secret — 설정 분리의 원리](./deployment-operations/04-configmap-secret.md) | 환경변수 주입과 파일 마운트 방식의 차이(파일 마운트는 ConfigMap 변경 시 자동 반영, 환경변수는 파드 재시작 필요), Secret이 Base64이지 암호화가 아닌 이유(etcd 암호화 설정 별도), Sealed Secret / External Secrets Operator로 GitOps 환경에서 시크릿을 안전하게 관리하는 방법 |
| [05. 모니터링과 로깅 — Prometheus와 Loki 파이프라인](./deployment-operations/05-monitoring-logging.md) | Prometheus Operator가 ServiceMonitor 리소스로 스크랩 대상을 동적으로 관리하는 방식, `kubectl top pod`과 HPA Controller가 동일한 Metrics Server → metrics.k8s.io API를 각각 독립적으로 사용하는 구조, Loki가 파드 레이블을 기반으로 로그를 수집하는 방식, 로그 레벨 별 비용 트레이드오프 |

</details>

<br/>

### 🔹 Chapter 7: 고급 패턴과 운영 심화

> **핵심 질문:** Operator 패턴으로 Reconciliation Loop를 직접 구현하면 무엇을 얻는가? Istio의 Envoy Sidecar는 어떻게 모든 트래픽을 가로채는가?

<details>
<summary><b>Operator 패턴부터 멀티 클러스터 운영까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Operator 패턴 — CRD와 Reconciliation Loop](./advanced-patterns/01-operator-crd.md) | CRD(Custom Resource Definition)로 쿠버네티스 API를 도메인 특화 리소스로 확장하는 방법, Custom Controller가 CR의 Desired State와 Actual State를 비교해 외부 시스템(DB 클러스터, 메시지 큐)을 관리하는 Reconciliation Loop 구현, controller-runtime 라이브러리로 Operator를 작성하는 구조 |
| [02. Admission Webhook — 리소스 생성/수정 인터셉트](./advanced-patterns/02-admission-webhook.md) | Mutating Webhook(리소스 스펙 자동 수정)과 Validating Webhook(유효성 검사 후 거부 가능)이 API Server Admission 파이프라인에서 호출되는 순서, Istio가 `istio-injection=enabled` 레이블이 있는 네임스페이스에 Envoy Sidecar를 자동 주입하는 방식, TLS 인증서와 `caBundle` 설정의 중요성 |
| [03. Service Mesh — Istio와 Envoy Sidecar](./advanced-patterns/03-service-mesh-istio.md) | Envoy Sidecar가 iptables `ISTIO_REDIRECT` 규칙으로 파드의 모든 인바운드/아웃바운드 트래픽을 가로채는 방식, Istiod(Control Plane)가 Envoy에 xDS 프로토콜로 라우팅 정보를 배포하는 구조, mTLS가 Envoy 간 자동으로 적용되는 원리, VirtualService로 Canary / Blue-Green 트래픽 분산 구현 |
| [04. etcd 운영 — 백업과 성능](./advanced-patterns/04-etcd-operations.md) | `etcdctl snapshot save`로 etcd 스냅샷을 생성하고 `etcdctl snapshot restore`로 복구하는 절차, etcd 클러스터에 멤버를 추가/제거하는 방법, 디스크 I/O가 Raft Log 복제 지연(fsync 시간)을 통해 Control Plane 전체 성능을 결정하는 이유, `etcd_disk_wal_fsync_duration_seconds` 메트릭 모니터링 |
| [05. 멀티 클러스터 운영 — 네트워킹과 트래픽 라우팅](./advanced-patterns/05-multi-cluster.md) | 멀티 클러스터가 필요한 이유(지역 분산 / 장애 격리 / 규정 준수), Submariner로 클러스터 간 파드 네트워크를 연결하는 방식, Istio Multi-Cluster로 서비스 메시를 클러스터 경계를 넘어 확장하는 방법, 지역(Region) 간 트래픽 라우팅과 Failover 설계 패턴 |

</details>

---

## 🏗️ 실험 환경

모든 실험은 Kind 기반 로컬 3노드 클러스터에서 재현할 수 있습니다.

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: false   # kindnet(기본 CNI) 사용 — Ch3-02 Flannel/Calico 실험 시 true로 변경 후 별도 CNI 설치 필요
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
```

```bash
# 클러스터 생성
kind create cluster --config kind-config.yaml --name deep-dive

# 실험용 공통 명령어 세트

# etcd 직접 조회 — 클러스터 상태 원본 데이터 확인
kubectl exec -n kube-system etcd-deep-dive-control-plane -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  get /registry/pods/default --prefix --keys-only

# iptables 규칙 확인 — kube-proxy가 생성한 Service 라우팅 규칙
kubectl exec -it -n kube-system kube-proxy-<hash> -- iptables-save | grep <service-name>
iptables -t nat -L KUBE-SERVICES -n --line-numbers

# API Server 이벤트 추적 — 파드 생성 타임라인 확인
kubectl get events --sort-by='.metadata.creationTimestamp' -A

# 파드 스케줄링 결정 추적
kubectl describe pod <pod-name> | grep -A 20 Events

# containerd 레벨 컨테이너 상태 확인
crictl ps
crictl inspect <container-id>

# cgroups 리소스 제한 확인
cat /sys/fs/cgroup/memory/kubepods/pod<pod-uid>/memory.limit_in_bytes

# 네트워크 정책 적용 확인
kubectl exec -it <source-pod> -- curl <target-pod-ip>:<port>
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 실무에서 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — 내부를 모를 때의 접근과 그 결과 |
| ✨ **올바른 접근** | After — 원리를 알고 난 후의 설계/운영 |
| 🔬 **내부 동작 원리** | 컴포넌트 소스 레벨 분석 + kubectl 이벤트 추적 |
| 💻 **실전 실험** | `kubectl describe`, `iptables-save`, `etcdctl get`, `crictl` 등으로 재현 |
| 📊 **성능/비용 비교** | iptables vs IPVS, Flannel vs Calico, emptyDir vs PV 등 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "CrashLoopBackOff 원인을 못 찾겠다" — 장애 대응 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch1-05  kubelet과 Probe → 컨테이너 상태 판단 기준 이해
Day 2  Ch2-01  파드 생성 전 과정 → 어느 단계에서 실패하는지 진단
       Ch2-04  파드 종료 → SIGTERM과 gracePeriod 이해
Day 3  Ch5-02  QoS 클래스 → OOMKilled 원인 분석
```

</details>

<details>
<summary><b>🟡 "Control Plane 내부를 제대로 이해하고 싶다" — 핵심 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch1-01  클러스터 아키텍처 개요 → 전체 그림 파악
       Ch1-02  API Server → 모든 요청의 진입점
Day 2  Ch1-03  etcd와 Raft → 클러스터 일관성의 근거
       Ch1-04  Controller Manager → Reconciliation Loop 실체
Day 3  Ch2-01  파드 생성 전 과정 → 5단계 전체 추적
       Ch2-02  스케줄링 알고리즘 → Filtering/Scoring 이해
Day 4  Ch3-03  Service와 kube-proxy → iptables 규칙 직접 확인
       Ch3-04  Service 종류 완전 분해
Day 5  Ch5-01  Request와 Limit → cgroups 구현 확인
       Ch5-02  QoS 클래스 → OOM 종료 순서
Day 6  Ch6-01  롤링 업데이트 알고리즘
       Ch6-02  무중단 배포 삼각편대
Day 7  Ch7-01  Operator 패턴 → Reconciliation Loop 직접 구현
```

</details>

<details>
<summary><b>🔴 "쿠버네티스 소스코드 레벨까지 완전 정복" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — Control Plane 아키텍처
        → etcdctl로 /registry 원본 데이터 확인, API Server --v=8 로그 분석

2주차  Chapter 2 전체 — 파드 생명주기
        → kubectl get events 타임라인 추적, crictl로 containerd 레벨 확인

3주차  Chapter 3 전체 — 네트워킹 완전 분해
        → iptables-save로 KUBE-SERVICES 체인 분석, CNI 플러그인 동작 비교

4주차  Chapter 4 전체 — 스토리지
        → 동적 프로비저닝 전 과정 추적, CSI Node Plugin 마운트 단계 확인

5주차  Chapter 5 전체 — 자원 관리와 스케일링
        → cgroups 파일 직접 확인, HPA 스케일 알고리즘 시뮬레이션

6주차  Chapter 6 전체 — 배포 전략과 운영
        → 무중단 배포 실험(preStop Hook 없을 때 연결 끊김 재현), RBAC 최소 권한 설계

7주차  Chapter 7 전체 — 고급 패턴
        → controller-runtime으로 간단한 Operator 작성, Envoy Sidecar 트래픽 가로채기 확인
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [docker-deep-dive](https://github.com/dev-book-lab/docker-deep-dive) | 컨테이너 namespace / cgroups / Union FS 원리 | Ch1-05(kubelet-CRI), Ch2-03(containerd-runc), Ch5-01(cgroups로 Limit 구현) |
| [linux-for-backend-deep-dive](https://github.com/dev-book-lab/linux-for-backend-deep-dive) | iptables / namespace / cgroups 커널 레벨 원리 | Ch3-03(kube-proxy iptables 규칙), Ch3-07(Network Policy eBPF), Ch5-01(cgroups cpu.cfs_quota_us) |
| [network-deep-dive](https://github.com/dev-book-lab/network-deep-dive) | L4/L7 로드밸런싱, VXLAN, BGP 라우팅 원리 | Ch3-02(Flannel VXLAN vs Calico BGP), Ch3-03(IPVS 모드), Ch3-05(Ingress L7 라우팅) |
| [redis-deep-dive](https://github.com/dev-book-lab/redis-deep-dive) | Redis 단일 스레드 이벤트 루프, 영속성, 복제 | Ch4-04(StatefulSet으로 Redis Cluster 운영), Ch6-04(ConfigMap으로 redis.conf 관리) |

> 💡 **선행 학습 권장 순서:** docker-deep-dive → linux-for-backend-deep-dive → network-deep-dive → **kubernetes-deep-dive**  
> docker-deep-dive를 마쳤다면 바로 시작할 수 있습니다. linux-deep-dive와 network-deep-dive의 개념은 해당 챕터에서 필요한 부분을 연결해서 설명합니다.

---

## 🙏 Reference

- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [Kubernetes in Action, 2nd Edition — Marko Luksa](https://www.manning.com/books/kubernetes-in-action-second-edition)
- [Programming Kubernetes — Michael Hausenblas & Stefan Schimanski](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)
- [Kubernetes 소스 코드 (GitHub)](https://github.com/kubernetes/kubernetes)
- [Learnk8s Blog](https://learnk8s.io/blog) — 쿠버네티스 내부 동작 시각화
- [Ivan Velichko's Blog (iximiuz)](https://iximiuz.com/en/) — 컨테이너/쿠버네티스 내부 심층 분석

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

**"kubectl apply로 파드를 배포하는 것과, Control Plane이 수천 개의 파드 상태를 어떻게 선언적으로 수렴시키는지 아는 것은 다르다"**

</div>
