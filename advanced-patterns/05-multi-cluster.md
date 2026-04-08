# 멀티 클러스터 운영 — 네트워킹과 트래픽 라우팅

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 멀티 클러스터가 필요한 상황은 어떤 경우이고, 언제는 불필요한가?
- Submariner는 어떻게 두 클러스터 간 파드 네트워크를 직접 연결하는가?
- Istio Multi-Cluster에서 서비스 메시가 클러스터 경계를 어떻게 넘는가?
- 지역(Region) 간 트래픽 Failover를 구현하는 방법은?
- 멀티 클러스터 운영에서 가장 자주 발생하는 장애 패턴은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"ap-northeast-2 리전이 다운되면 서비스가 멈춘다"는 문제는 단일 클러스터 아키텍처의 근본 한계다. 멀티 클러스터는 이 한계를 극복하지만, 구현 복잡도가 크게 증가한다. 무엇이 클러스터 경계를 넘을 수 있는지, 어떤 기술이 필요한지 이해해야 올바른 아키텍처를 설계할 수 있다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 두 클러스터를 만들었는데 서비스가 서로 통신 안 됨

원리를 모를 때의 판단:
  "같은 VPC에 있으니 통신 되어야 하는 것 아닌가?"
  → VPC 피어링 확인 → 정상
  → 여전히 안 됨

실제 원인:
  VPC 피어링은 노드 간 통신만 해결
  파드 IP는 각 클러스터가 독립적으로 관리 (CIDR 겹침 가능)
  
  클러스터 A: 파드 CIDR 10.244.0.0/16
  클러스터 B: 파드 CIDR 10.244.0.0/16  ← 동일!
  → 라우팅 불가 (CIDR 충돌)

해결:
  클러스터 설계 시 CIDR을 겹치지 않게:
    클러스터 A: 파드 CIDR 10.244.0.0/16
    클러스터 B: 파드 CIDR 10.245.0.0/16
  
  그 후 Submariner 또는 VPN으로 연결
```

---

## ✨ 올바른 접근 (After — 멀티 클러스터 필요성 판단과 아키텍처)

```
멀티 클러스터 필요한 경우:
  지역 분산 HA: 리전 장애 시 자동 Failover
  규정 준수: 데이터가 특정 국가/리전을 벗어나면 안 됨
  장애 격리: 클러스터 하나가 망가져도 다른 클러스터 영향 없음
  환경 분리: 개발/스테이징/프로덕션 완전 분리

멀티 클러스터 불필요한 경우:
  단일 리전으로 충분한 서비스
  팀이 멀티 클러스터 운영 경험 없음
  → 하나의 클러스터에서 고가용성 (Multi-AZ 노드 그룹)으로 충분

기술 선택:
  네트워크 연결: Submariner, Cilium Cluster Mesh, WireGuard VPN
  서비스 디스커버리: Istio Multi-Cluster, KubeFed
  글로벌 로드밸런싱: AWS Route53 Failover, Cloudflare Load Balancing
  GitOps 다중 배포: ArgoCD ApplicationSet, Flux Fleets
```

---

## 🔬 내부 동작 원리

### 멀티 클러스터 아키텍처 패턴

```
패턴 1: 독립 클러스터 + 글로벌 LB 기반 Failover
  
  사용자 요청
    ↓
  Route53 / Cloudflare LB
    ├── 클러스터 A (us-east-1) → 활성
    └── 클러스터 B (ap-northeast-2) → 대기 (Primary 장애 시 전환)
  
  클러스터 간 직접 통신 없음 (독립 운영)
  단점: 클러스터 간 데이터 동기화 문제 (DB)

패턴 2: 연결된 클러스터 (Cluster Mesh)

  클러스터 A ←─ Submariner / Cilium ─→ 클러스터 B
  파드 IP 직접 통신 가능
  서비스 디스커버리 공유
  단점: 네트워크 설정 복잡, 레이턴시 증가

패턴 3: 컨트롤 플레인 분리 + 데이터 플레인 공유

  중앙 컨트롤 플레인 (Anthos, KubeVela)
    → 여러 클러스터/리전에 워크로드 배포
  각 클러스터는 독립적으로 실행
```

### Submariner — 클러스터 간 파드 네트워크 연결

```
Submariner 컴포넌트:
  Gateway: 각 클러스터에 1개, 클러스터 간 VPN 터널 관리
  Route Agent: 각 노드에 DaemonSet, 파드 IP 라우팅 테이블 관리
  Service Discover: ServiceExport/ServiceImport CRD

동작 원리:
  1. 각 클러스터에 Submariner 설치
  
  2. Gateway 간 IPsec/WireGuard 터널 수립
     클러스터 A Gateway (203.0.113.1)
     클러스터 B Gateway (203.0.113.2)
     → 암호화된 터널로 연결
  
  3. Route Agent가 각 노드에 라우팅 규칙 추가:
     클러스터 A 노드:
       ip route add 10.245.0.0/16 via <Gateway A IP>
       (클러스터 B 파드 CIDR → Gateway로 전달)
  
  4. 파드 A (10.244.1.5) → 파드 B (10.245.1.7):
     클러스터 A 노드 → Gateway A → IPsec 터널 → Gateway B → 클러스터 B 노드 → 파드 B

ServiceImport/ServiceExport:
  클러스터 A에서 서비스 내보내기:
    kubectl create serviceexport my-service -n production
  
  클러스터 B에서 자동으로 ServiceImport 생성:
    my-service.production.svc.clusterset.local
    → 클러스터 A의 파드들로 라우팅
```

### Istio Multi-Cluster — 서비스 메시 확장

```
Primary-Remote 설정:
  Primary 클러스터: Istiod 실행 (Control Plane)
  Remote 클러스터: Envoy만 실행, Primary Istiod에 연결
  
  Istiod가 두 클러스터 모두의 서비스 디스커버리를 담당:
    클러스터 A의 서비스 정보 → 클러스터 B의 Envoy에도 전달
    → 클러스터 B의 파드가 클러스터 A의 서비스를 알 수 있음

멀티 Primary 설정:
  각 클러스터에 Istiod 실행
  두 Istiod가 서로의 클러스터 정보를 동기화
  더 높은 가용성 (하나의 클러스터가 Istiod 없어도 동작)

VirtualService로 클러스터 간 트래픽:
  spec:
    hosts:
    - my-service
    http:
    - route:
      - destination:
          host: my-service
          subset: cluster-a
        weight: 50
      - destination:
          host: my-service
          subset: cluster-b
        weight: 50
```

### 지역 간 Failover 설계

```
패턴: Active-Active (두 리전 모두 활성)

  사용자 → Route53 (레이턴시 기반 라우팅)
             → 가장 가까운 리전으로

  헬스체크 + Failover:
    Route53이 각 리전의 ALB를 주기적으로 헬스체크
    리전 A 장애 감지 → 리전 B로 자동 전환
    TTL을 낮게 (30초) 설정하여 빠른 전환

패턴: Active-Passive (기본 리전 + 대기 리전)

  평상시: 리전 A만 트래픽 처리
  장애 시: Route53 Failover → 리전 B 활성화
  RTO(복구 목표 시간): 30초~5분 (TTL + 전환 시간)
  RPO(데이터 손실 허용): DB 복제 지연에 따라 다름

DB 데이터 동기화:
  RDS Multi-Region Read Replica + Promotion
  CockroachDB: 다중 리전 분산 DB 기본 지원
  DynamoDB Global Tables: 자동 다중 리전 복제
```

---

## 💻 실전 실험

### 1. 두 개의 Kind 클러스터 생성

```bash
# 클러스터 A (CIDR: 10.244.0.0/16)
cat <<EOF > cluster-a.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
nodes:
- role: control-plane
- role: worker
EOF
kind create cluster --name cluster-a --config cluster-a.yaml

# 클러스터 B (CIDR: 10.245.0.0/16)
cat <<EOF > cluster-b.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  podSubnet: "10.245.0.0/16"
  serviceSubnet: "10.97.0.0/12"
nodes:
- role: control-plane
- role: worker
EOF
kind create cluster --name cluster-b --config cluster-b.yaml
```

### 2. ArgoCD ApplicationSet으로 멀티 클러스터 배포

```bash
# ArgoCD 설치
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 두 번째 클러스터 등록
argocd cluster add kind-cluster-b --name cluster-b

# ApplicationSet으로 두 클러스터에 동시 배포
cat <<EOF | kubectl apply -n argocd -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-multicluster
spec:
  generators:
  - clusters: {}   # 등록된 모든 클러스터
  template:
    metadata:
      name: '{{name}}-my-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/k8s-manifests
        targetRevision: HEAD
        path: apps/my-app
      destination:
        server: '{{server}}'
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
EOF
```

### 3. Cilium Cluster Mesh (Kind)

```bash
# Cilium으로 두 클러스터 설치
helm install cilium cilium/cilium --namespace kube-system \
  --kube-context kind-cluster-a \
  --set cluster.name=cluster-a \
  --set cluster.id=1 \
  --set clustermesh.useAPIServer=true

helm install cilium cilium/cilium --namespace kube-system \
  --kube-context kind-cluster-b \
  --set cluster.name=cluster-b \
  --set cluster.id=2 \
  --set clustermesh.useAPIServer=true

# Cluster Mesh 연결
cilium clustermesh connect \
  --context kind-cluster-a \
  --destination-context kind-cluster-b

# 글로벌 서비스 (두 클러스터 모두의 파드에 로드밸런싱)
kubectl annotate service my-service \
  service.cilium.io/global="true"
```

---

## 📊 멀티 클러스터 기술 비교

| 기술 | 파드 간 직접 통신 | 서비스 디스커버리 | mTLS | 설정 복잡도 |
|-----|--------------|--------------|-----|----------|
| Submariner | ✅ (IPsec) | ✅ (ServiceImport) | 터널 레벨 | 중간 |
| Cilium Cluster Mesh | ✅ (eBPF) | ✅ (Global Service) | ✅ | 중간 |
| Istio Multi-Cluster | ✅ (East-West Gateway) | ✅ (ServiceEntry) | ✅ | 높음 |
| 독립 + Route53 | ❌ | ❌ | ❌ | 낮음 |

---

## ⚖️ 트레이드오프

**운영 복잡도 vs 가용성**

멀티 클러스터는 운영 복잡도를 크게 높인다. 배포, 모니터링, 장애 대응이 모두 클러스터 수만큼 배가된다. "2개 클러스터 = 2배 운영 부담"이 아니라 네트워크 연결, CIDR 관리, 인증서 동기화 등의 추가 복잡성이 더해져 실제로는 3~4배 부담이 될 수 있다. 단일 클러스터에서 Multi-AZ 고가용성으로 충분한지 먼저 검토해야 한다.

**레이턴시 vs 가용성**

Active-Active 멀티 리전 아키텍처는 지역 장애에 대한 최고의 가용성을 제공하지만, 리전 간 데이터 동기화 레이턴시가 존재한다. 특히 RDBMS에서 리전 간 동기식 복제는 모든 쓰기에 리전 간 왕복 시간(50~200ms)이 추가된다. 비동기 복제는 레이턴시 영향이 없지만 장애 시 데이터 손실(RPO > 0)이 발생할 수 있다.

---

## 📌 핵심 정리

```
멀티 클러스터 필요 조건:
  리전 장애 대응, 데이터 주권, 환경 격리
  단순 고가용성은 단일 클러스터 Multi-AZ로 충분

설계 전제조건:
  클러스터 간 파드 CIDR 겹치지 않게
  서비스 CIDR도 겹치지 않게

네트워크 연결:
  Submariner: IPsec 터널, ServiceImport/Export CRD
  Cilium Cluster Mesh: eBPF, Global Service 어노테이션
  Istio Multi-Cluster: East-West Gateway, xDS 동기화

Failover:
  Route53/Cloudflare: DNS TTL 낮춤 + 헬스체크 기반
  Active-Active: 두 리전 모두 활성, 레이턴시 기반 라우팅
  Active-Passive: Primary 장애 시 Standby 활성화

멀티 클러스터 배포:
  ArgoCD ApplicationSet: 등록된 모든 클러스터에 동시 배포
  Flux Fleets: GitOps 다중 클러스터 관리
```

---

## 🤔 생각해볼 문제

**Q1.** Active-Active 멀티 리전 아키텍처에서 사용자의 세션(로그인 상태)을 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

세션 데이터를 공유 저장소에 유지해야 한다. 리전 A에서 로그인한 사용자가 Route53에 의해 리전 B로 라우팅되면, 리전 B의 서버도 해당 세션 정보를 알아야 한다. 해결 방법은 (1) 세션을 JWT로 클라이언트에 저장 (stateless, 권장), (2) Redis Global Database (ElastiCache Global Datastore, 리전 간 복제)에 세션 저장, (3) 세션 어피니티(Sticky Session) - 한 번 연결된 리전으로만 라우팅 (하지만 리전 장애 시 세션 손실). JWT 방식이 가장 단순하고 확장성 있다.

</details>

**Q2.** 두 클러스터가 Submariner로 연결되어 있을 때, 한 클러스터의 파드가 다른 클러스터의 파드 IP로 직접 통신하는 것이 보안상 문제가 되는가?

<details>
<summary>해설 보기</summary>

Submariner는 클러스터 간 IPsec 암호화 터널을 사용하므로 네트워크 레벨 암호화는 제공된다. 그러나 어떤 파드가 어떤 파드에 접근할 수 있는지 제어하는 것은 별도의 Network Policy가 필요하다. Cilium이나 Calico의 Network Policy를 클러스터 간으로 확장하는 기능도 일부 제품에서 지원한다. 또한 Istio Multi-Cluster와 함께 사용하면 AuthorizationPolicy로 서비스 간 인가를 세밀하게 제어할 수 있다.

</details>

**Q3.** 멀티 클러스터에서 ConfigMap과 Secret을 어떻게 일관되게 관리하는가?

<details>
<summary>해설 보기</summary>

몇 가지 접근이 있다. (1) GitOps (ArgoCD/Flux): 모든 ConfigMap을 Git에 저장하고 각 클러스터에 동기화. 단, Secret은 Sealed Secret 또는 External Secrets로 처리. (2) External Secrets Operator: AWS SM, Vault 같은 중앙 비밀 저장소에서 각 클러스터가 독립적으로 Secret을 동기화. (3) ClusterAPI + KubeFed: 페더레이션 레이어에서 ConfigMap/Secret을 여러 클러스터에 배포. 실제로는 GitOps + ESO 조합이 가장 많이 사용된다.

</details>

---

<div align="center">

**[⬅️ 이전: etcd 운영 — 백업과 성능](./04-etcd-operations.md)** | **[홈으로 🏠](../README.md)**

</div>
