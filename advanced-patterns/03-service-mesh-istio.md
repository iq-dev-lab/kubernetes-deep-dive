# Service Mesh — Istio와 Envoy Sidecar

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Envoy Sidecar가 파드의 모든 트래픽을 가로채는 iptables 원리는?
- Istiod가 xDS 프로토콜로 Envoy에 라우팅 정보를 배포하는 구조는?
- mTLS가 애플리케이션 코드 수정 없이 자동으로 적용되는 원리는?
- VirtualService로 Canary/Blue-Green 트래픽을 어떻게 분산하는가?
- Istio 없이 직접 트래픽 제어가 필요할 때 어떤 대안이 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"서비스 간 인증(mTLS)을 전부 구현해야 하는데 코드는 건드리기 싫다"거나 "카나리 배포 중 10%만 새 버전으로 보내고 싶다"는 요구사항이 Service Mesh 없이는 구현이 매우 복잡하다. Istio는 이를 애플리케이션 코드 수정 없이 인프라 레벨에서 구현한다. 단, Istio 자체의 복잡성과 운영 부담도 상당하므로, 언제 쓰고 언제 쓰지 말아야 하는지를 이해해야 한다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: Istio 설치 후 파드 간 통신이 갑자기 안 됨

원리를 모를 때의 판단:
  "Istio 설치가 잘못됐나?" → 재설치
  → 동일 증상

실제 원인:
  Istio는 기본적으로 모든 서비스 간 mTLS를 자동 활성화
  기존 서비스 중 Istio Sidecar가 없는 파드가 있는 경우
  Sidecar 있는 서비스 → Sidecar 없는 서비스: mTLS 핸드셰이크 실패

진단:
  kubectl exec <pod> -c istio-proxy -- \
    pilot-agent request GET /config_dump | grep -i tls

  PeerAuthentication 설정 확인:
    kubectl get peerauthentication -A
  
  해결:
    PeerAuthentication을 PERMISSIVE 모드로 설정 (mTLS + 평문 허용)
    또는 모든 파드에 Sidecar 주입
```

---

## ✨ 올바른 접근 (After — Service Mesh 아키텍처 이해)

```
Istio 도입 전 확인할 필요사항:

필요성 확인:
  ✅ 서비스 간 mTLS 필요 (제로 트러스트 보안)
  ✅ 트래픽 세밀한 제어 (카나리, A/B 테스트)
  ✅ 분산 트레이싱 (서비스 간 요청 추적)
  ✅ 서킷 브레이커, Retry 정책
  → Istio 도입 적합

불필요한 경우:
  ❌ 서비스가 10개 미만 단순 아키텍처
  ❌ 팀에 Istio 운영 경험 없음
  → 서비스 레벨 mTLS, Nginx Ingress로 충분

대안:
  Linkerd: 경량 Service Mesh (Rust Sidecar, Istio 대비 낮은 리소스)
  Cilium Service Mesh: eBPF 기반 (Sidecar 없음)
  Consul Connect: HashiCorp 생태계
```

---

## 🔬 내부 동작 원리

### Envoy Sidecar iptables 트래픽 가로채기

```
istio-init Init Container 실행 (파드 시작 전):

  # 모든 인바운드 트래픽을 Envoy로 리다이렉션
  iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-port 15006

  # 모든 아웃바운드 트래픽을 Envoy로 리다이렉션
  iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-port 15001

  # Envoy 자신의 트래픽은 제외 (무한루프 방지, UID 1337)
  iptables -t nat -A OUTPUT -m owner --uid-owner 1337 -j RETURN

  # 쿠버네티스 API 서버 등 일부 IP 제외
  iptables -t nat -A OUTPUT -d 10.96.0.1 -j RETURN

트래픽 흐름 (인바운드):
  외부 요청 → 파드 IP:8080
    ↓ iptables PREROUTING
  Envoy 15006 (Inbound)
    ↓ mTLS 복호화, 메트릭 수집, 추적 헤더 추가
  localhost:8080 (실제 앱)

트래픽 흐름 (아웃바운드):
  앱 → 다른 서비스 URL
    ↓ iptables OUTPUT
  Envoy 15001 (Outbound)
    ↓ 서비스 디스커버리, 로드밸런싱, mTLS 암호화
  실제 대상 파드
```

### Istiod — Control Plane과 xDS 프로토콜

```
xDS (x Discovery Service) 프로토콜:
  Envoy가 동적으로 설정을 받아오는 프로토콜
  CDS (Cluster Discovery Service): 업스트림 클러스터 목록
  EDS (Endpoint Discovery Service): 각 클러스터의 엔드포인트
  LDS (Listener Discovery Service): 어떤 포트/프로토콜 Listen
  RDS (Route Discovery Service): 트래픽 라우팅 규칙

Istiod → Envoy xDS 흐름:
  1. Istiod가 k8s Service, Endpoints, VirtualService, DestinationRule Watch
  
  2. Envoy가 시작되면 Istiod에 gRPC 연결
     (ADS: Aggregated Discovery Service)
  
  3. Istiod가 현재 클러스터 서비스 구성 전송:
     - 어떤 서비스가 있는가 (CDS)
     - 각 서비스의 엔드포인트 IP:port (EDS)
     - 어떤 포트를 Listen할지 (LDS)
     - 트래픽 라우팅 규칙 (RDS)
  
  4. 변경 발생 시 Istiod가 변경된 설정 Push

Envoy 설정 덤프 확인:
  kubectl exec <pod> -c istio-proxy -- \
    pilot-agent request GET /config_dump
```

### mTLS 자동 적용

```
STRICT 모드 (PeerAuthentication):
  Sidecar 있는 서비스끼리만 통신 허용
  모든 연결에 mTLS 필수

  동작:
    서비스 A (Sidecar) → 서비스 B (Sidecar):
      Envoy A: TLS 핸드셰이크 시작 (인증서 제시)
      Envoy B: A의 인증서 검증 → 연결 수립
      통신: 암호화된 채널

  인증서 관리:
    Istiod가 각 ServiceAccount에 대한 X.509 인증서 발급
    SPIFFE(Secure Production Identity Framework For Everyone) 형식:
    spiffe://cluster.local/ns/production/sa/my-service
    Envoy에 자동 주입, 24시간마다 자동 갱신

PERMISSIVE 모드:
  mTLS + 평문 둘 다 허용
  마이그레이션 기간 중 사용 (일부 서비스는 Sidecar 없음)
```

### VirtualService로 트래픽 분산

```yaml
# 카나리 배포: v2로 20% 트래픽
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-service
spec:
  hosts:
  - my-service
  http:
  - route:
    - destination:
        host: my-service
        subset: v1        # DestinationRule에서 레이블로 정의
      weight: 80          # 80% → 기존 버전
    - destination:
        host: my-service
        subset: v2
      weight: 20          # 20% → 카나리 버전
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-service
spec:
  host: my-service
  subsets:
  - name: v1
    labels:
      version: v1         # 파드 레이블로 그룹화
  - name: v2
    labels:
      version: v2
```

```yaml
# 헤더 기반 라우팅 (특정 사용자만 새 버전)
http:
- match:
  - headers:
      x-user-group:
        exact: beta-tester
  route:
  - destination:
      host: my-service
      subset: v2          # beta-tester는 v2로
- route:
  - destination:
      host: my-service
      subset: v1          # 나머지는 v1으로
```

---

## 💻 실전 실험

### 1. Istio 설치 (Kind)

```bash
# istioctl 설치
curl -L https://istio.io/downloadIstio | sh -
export PATH=$PATH:$(pwd)/istio-*/bin

# Kind에 Istio 설치 (demo 프로파일)
istioctl install --set profile=demo -y

# 설치 확인
kubectl get pods -n istio-system
# istiod-xxx            1/1  Running   ← Control Plane
# istio-ingressgateway  1/1  Running   ← Gateway
```

### 2. Sidecar 주입 활성화 및 확인

```bash
# Namespace에 Sidecar 자동 주입 활성화
kubectl label namespace default istio-injection=enabled

# 테스트 파드 배포
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/httpbin/httpbin.yaml

# Sidecar 확인
kubectl get pod -l app=httpbin
# READY: 2/2 (앱 컨테이너 + istio-proxy)
```

### 3. VirtualService 카나리 배포

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: v1
  template:
    metadata:
      labels:
        app: my-app
        version: v1
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=v1"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      version: v2
  template:
    metadata:
      labels:
        app: my-app
        version: v2
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args: ["-text=v2"]
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app  # v1, v2 모두 선택
  ports:
  - port: 5678
EOF

# VirtualService로 20% 카나리
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-app
spec:
  host: my-app
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
  - my-app
  http:
  - route:
    - destination:
        host: my-app
        subset: v1
      weight: 80
    - destination:
        host: my-app
        subset: v2
      weight: 20
EOF

# 트래픽 분산 확인
for i in $(seq 20); do
  kubectl exec -it sleep-pod -- curl http://my-app:5678
done
# v1이 ~16개, v2가 ~4개 출력
```

---

## 📊 Service Mesh 구현 비교

| 항목 | Istio | Linkerd | Cilium SM |
|-----|------|--------|---------|
| Sidecar | Envoy (Go + C++) | 경량 Rust Proxy | 없음 (eBPF) |
| 리소스 사용 | 높음 | 낮음 | 매우 낮음 |
| mTLS | ✅ 자동 | ✅ 자동 | ✅ |
| 트래픽 제어 | VirtualService (풍부) | HTTPRoute | 제한적 |
| 학습 곡선 | 높음 | 중간 | 중간 |
| 성숙도 | 매우 높음 | 높음 | 높음 |

---

## ⚖️ 트레이드오프

**Sidecar 오버헤드**

각 파드마다 Envoy Sidecar가 추가되어 파드당 약 50m CPU, 100MB 메모리가 소비된다. 파드 100개면 추가 5 CPU, 10GB 메모리가 필요하다. 또한 모든 요청이 Envoy를 두 번 통과(출발지 Envoy + 목적지 Envoy)하므로 레이턴시가 수 ms 증가한다. 대부분의 경우 무시할 수 있는 수준이지만, 레이턴시에 극도로 민감한 시스템에서는 고려가 필요하다.

**Istio 복잡성**

VirtualService, DestinationRule, Gateway, PeerAuthentication, AuthorizationPolicy... Istio의 CRD는 수십 개에 달한다. 이 설정들이 상호작용하는 방식을 이해하는 데 상당한 학습 시간이 필요하다. 소규모 팀이나 단순한 아키텍처에서는 Istio 도입이 오히려 운영 복잡성을 크게 높일 수 있다.

---

## 📌 핵심 정리

```
Envoy 트래픽 가로채기:
  istio-init: iptables PREROUTING/OUTPUT → 모든 트래픽을 Envoy로
  Envoy UID 1337: 자신의 트래픽은 iptables 제외 (무한루프 방지)

Istiod xDS:
  k8s 리소스 Watch → Envoy에 gRPC로 동적 설정 배포
  CDS/EDS/LDS/RDS → 서비스 목록, 엔드포인트, 라우팅 규칙

mTLS 자동화:
  SPIFFE X.509 인증서 자동 발급/갱신 (Istiod)
  STRICT: mTLS 필수, PERMISSIVE: mTLS + 평문 허용

VirtualService 트래픽 분산:
  weight 기반: 카나리 (80%/20%)
  헤더 기반: A/B 테스트, 특정 사용자 라우팅
  DestinationRule의 subset으로 파드 그룹 정의
```

---

## 🤔 생각해볼 문제

**Q1.** 파드 A(Sidecar 있음)에서 파드 B(Sidecar 없음)로 요청할 때, mTLS STRICT 모드면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

파드 A의 Envoy가 파드 B로 mTLS 연결을 시도한다. 파드 B에는 Envoy가 없으므로 TLS 핸드셰이크를 처리할 수 없다. 연결이 실패하고 파드 A의 Envoy는 503 오류를 반환한다. 해결 방법은 세 가지다. (1) 파드 B에도 Sidecar를 주입한다. (2) PeerAuthentication을 PERMISSIVE 모드로 변경한다. (3) DestinationRule에서 파드 B에 대한 TLS 설정을 비활성화한다(`trafficPolicy.tls.mode: DISABLE`).

</details>

**Q2.** Istio가 없을 때 쿠버네티스에서 카나리 배포를 구현하는 방법은?

<details>
<summary>해설 보기</summary>

몇 가지 방법이 있다. (1) 레플리카 수 기반: v1 파드 9개, v2 파드 1개를 같은 Service에 연결하면 10% 트래픽이 v2로 간다. 단, 정확한 비율 제어가 어렵다. (2) 두 개의 Deployment + 두 개의 Service + Nginx Ingress: Ingress의 `canary-weight` 어노테이션으로 Nginx 레벨에서 가중치 라우팅. (3) Argo Rollouts: 쿠버네티스 네이티브 카나리/Blue-Green 배포 컨트롤러로, Istio 없이도 정밀한 트래픽 제어 가능. (4) Flagger: HPA와 통합된 자동 카나리 분석 및 프로모션 도구.

</details>

**Q3.** 분산 트레이싱(Jaeger/Zipkin)이 Istio와 통합될 때 애플리케이션 코드에서 해야 하는 것은?

<details>
<summary>해설 보기</summary>

Istio의 Envoy는 각 요청에 `x-request-id`, `x-b3-traceid`, `x-b3-spanid` 등 트레이싱 헤더를 자동으로 추가하고 전파한다. 단, 이 헤더가 서비스 A → 서비스 B → 서비스 C를 거칠 때 올바르게 전파되려면, 각 서비스가 인바운드 요청의 트레이싱 헤더를 아웃바운드 요청에 그대로 포함해야 한다. 이 헤더 전파는 애플리케이션 코드에서 구현해야 한다. Envoy는 자신이 관여하는 hop만 기록할 수 있고, 서비스 내부 전파는 알 수 없다. Spring Boot의 경우 Spring Cloud Sleuth, Node.js는 opentelemetry SDK로 자동화할 수 있다.

</details>

---

<div align="center">

**[⬅️ 이전: Admission Webhook — 리소스 생성/수정 인터셉트](./02-admission-webhook.md)** | **[홈으로 🏠](../README.md)** | **[다음: etcd 운영 — 백업과 성능 ➡️](./04-etcd-operations.md)**

</div>
