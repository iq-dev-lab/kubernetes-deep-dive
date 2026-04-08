# CoreDNS — Service 이름 해석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 파드 내에서 `my-service`만 입력해도 ClusterIP가 해석되는 원리는 무엇인가?
- `<service>.<namespace>.svc.cluster.local` 형식은 어떻게 결정되는가?
- `/etc/resolv.conf`의 `ndots:5` 설정은 DNS 조회 횟수에 어떤 영향을 미치는가?
- CoreDNS는 어떻게 구성되고, 커스터마이징은 어디서 하는가?
- `kubectl exec`으로 파드 내 DNS 조회 과정을 어떻게 추적하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이크로서비스 환경에서 서비스 간 호출은 대부분 DNS 이름으로 이루어진다. "DB에 연결이 안 된다"는 문제의 상당수가 DNS 이름 오타, 네임스페이스 불일치, CoreDNS 파드 장애에서 비롯된다. 또한 `ndots:5` 설정 때문에 외부 도메인 하나를 조회할 때 실제로는 5번의 DNS 쿼리가 발생하는 현상이 있다. 대규모 클러스터에서 CoreDNS CPU 사용량이 급증하는 원인 중 하나다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 파드에서 다른 네임스페이스의 서비스에 접근이 안 됨
      http://user-service/api  → connection refused

원리를 모를 때의 판단:
  "user-service라고 하면 되지 않나?" → 안 됨
  "방화벽 문제인가?" → 확인 → 정상
  → 원인 모름

실제 원인:
  kubectl run test --image=busybox -- nslookup user-service
  → 현재 네임스페이스(default)에서만 찾음
  
  user-service가 'users' 네임스페이스에 있는 경우:
  올바른 DNS 이름: user-service.users.svc.cluster.local
  또는 단축: user-service.users  (같은 클러스터)

DNS 이름 규칙:
  같은 네임스페이스: my-service (또는 my-service.default)
  다른 네임스페이스: my-service.other-namespace.svc.cluster.local
```

---

## ✨ 올바른 접근 (After — CoreDNS 구조를 이해한 운영)

```
쿠버네티스 DNS FQDN 형식:
  Service: <service-name>.<namespace>.svc.cluster.local
  Pod:     <pod-ip>.<namespace>.pod.cluster.local
            (예: 10-244-1-5.default.pod.cluster.local)

단축 형식 (같은 네임스페이스):
  my-service           → my-service.default.svc.cluster.local
  my-service.prod      → my-service.prod.svc.cluster.local

StatefulSet Pod DNS:
  my-pod-0.my-service.namespace.svc.cluster.local
  my-pod-1.my-service.namespace.svc.cluster.local

DNS 조회 우선순위 (ndots:5 규칙):
  "."이 5개 미만이면 search domain을 붙여서 시도
  → 짧은 이름은 자동으로 클러스터 DNS로 시도
```

---

## 🔬 내부 동작 원리

### CoreDNS 아키텍처

CoreDNS는 kube-system 네임스페이스에 Deployment로 실행되며, ClusterIP Service로 노출된다.

```bash
kubectl get svc -n kube-system kube-dns
# NAME       TYPE        CLUSTER-IP   PORT(S)         AGE
# kube-dns   ClusterIP   10.96.0.10   53/UDP,53/TCP   10d

# 파드의 /etc/resolv.conf에 이 IP가 nameserver로 설정됨
kubectl exec test-pod -- cat /etc/resolv.conf
# nameserver 10.96.0.10      ← CoreDNS Service IP
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

CoreDNS는 플러그인 기반 아키텍처로 동작한다. 각 DNS 쿼리는 Corefile에 정의된 플러그인 체인을 순서대로 통과한다.

```
CoreDNS Corefile (kube-system의 ConfigMap coredns):
  .:53 {
      errors              ← 에러 로깅
      health {
         lameduck 5s      ← graceful shutdown
      }
      ready               ← 준비 상태 엔드포인트
      kubernetes cluster.local in-addr.arpa ip6.arpa {
         pods insecure    ← Pod DNS 활성화
         fallthrough in-addr.arpa ip6.arpa
         ttl 30           ← DNS 응답 TTL 30초
      }
      prometheus :9153    ← 메트릭 노출
      forward . /etc/resolv.conf {  ← 외부 도메인은 노드 DNS로 포워딩
         max_concurrent 1000
      }
      cache 30            ← 응답 캐시 30초
      loop                ← 루프 감지
      reload              ← Corefile 변경 자동 반영
      loadbalance         ← DNS 응답 레코드 순서 랜덤화
  }
```

### DNS 쿼리 처리 흐름

```
파드 내에서 my-service 조회:

  1. 파드 → DNS 쿼리: my-service (A 레코드)
     → nameserver: 10.96.0.10 (CoreDNS)

  2. CoreDNS: kubernetes 플러그인 처리
     → kube-dns API 조회: my-service가 ClusterIP Service인가?
     → Yes: 10.96.43.125 반환 (ClusterIP)

  3. 파드가 10.96.43.125 수신
     → iptables DNAT → 실제 파드로 전달
```

### `ndots:5` 설정의 영향

`ndots:5`는 DNS 이름에 점(`.`)이 5개 미만이면 search domain을 붙여서 시도하라는 의미다.

```
예시: curl http://api.external-company.com 조회 시
      api.external-company.com → 점 2개 < 5 → search domain 먼저 시도

실제 발생하는 DNS 쿼리 순서 (search: default.svc.cluster.local svc.cluster.local cluster.local):
  1. api.external-company.com.default.svc.cluster.local → NXDOMAIN
  2. api.external-company.com.svc.cluster.local         → NXDOMAIN
  3. api.external-company.com.cluster.local             → NXDOMAIN
  4. api.external-company.com (FQDN)                   → 성공

→ 외부 도메인 1개를 조회하는 데 4번의 DNS 쿼리 발생!

마이크로서비스가 외부 API를 자주 호출하면:
  CoreDNS에 불필요한 NXDOMAIN 쿼리 폭증
  → CoreDNS CPU 사용량 급증
  → DNS 응답 지연

해결 방법 1: 도메인 끝에 점 추가 (FQDN 명시)
  curl http://api.external-company.com.  ← 끝에 점 → search domain 스킵 → 즉시 외부 쿼리

해결 방법 2: /etc/resolv.conf의 ndots 줄이기
  파드 스펙에 dnsConfig 설정:
  spec:
    dnsConfig:
      options:
      - name: ndots
        value: "1"

해결 방법 3: CoreDNS에 autopath 플러그인 활성화
  첫 번째 쿼리에서 모든 search domain을 서버 측에서 시도하고 첫 결과 반환
  클라이언트 왕복 횟수 감소
```

### 파드 DNS 설정 커스터마이징

```yaml
spec:
  dnsPolicy: ClusterFirst   # 기본값: 클러스터 DNS 우선
  dnsConfig:
    nameservers:            # 추가 DNS 서버
    - 1.1.1.1
    searches:               # 추가 search domain
    - my-company.internal
    options:
    - name: ndots
      value: "2"            # ndots 낮춰 외부 DNS 쿼리 최적화
    - name: timeout
      value: "5"
```

```
dnsPolicy 옵션:
  ClusterFirst (기본): 클러스터 DNS 우선, 실패 시 업스트림
  ClusterFirstWithHostNet: hostNetwork 파드에서 ClusterFirst 사용
  Default: 노드의 /etc/resolv.conf 사용 (클러스터 DNS 안 씀)
  None: dnsConfig 설정 완전 직접 지정
```

### CoreDNS Configmap 커스터마이징

```bash
# CoreDNS 설정 수정
kubectl edit configmap coredns -n kube-system

# 예: 특정 도메인을 외부 DNS로 포워딩
# 기존 forward 블록에 추가:
# forward company.internal 10.0.0.1
```

---

## 💻 실전 실험

### 1. DNS 쿼리 계층 확인

```bash
# 테스트 파드 생성
kubectl run dns-debug --image=busybox --rm -it -- sh

# /etc/resolv.conf 확인
cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5

# 단계별 DNS 조회 테스트
# 같은 네임스페이스 (단축 이름)
nslookup kubernetes
# Server: 10.96.0.10
# Address: 10.96.x.x (kubernetes Service ClusterIP)

# 다른 네임스페이스
nslookup kube-dns.kube-system
# → kube-dns.kube-system.svc.cluster.local 해석

# FQDN
nslookup kube-dns.kube-system.svc.cluster.local
```

### 2. ndots:5로 인한 과도한 DNS 쿼리 관찰

```bash
# CoreDNS 로그에서 쿼리 관찰
kubectl logs -n kube-system -l k8s-app=kube-dns -f &

# 테스트 파드에서 외부 도메인 조회
kubectl exec dns-debug -- wget -q http://example.com -O /dev/null

# CoreDNS 로그 출력 (NXDOMAIN 쿼리 확인):
# [INFO] 10.244.x.x - "A IN example.com.default.svc.cluster.local" NXDOMAIN
# [INFO] 10.244.x.x - "A IN example.com.svc.cluster.local" NXDOMAIN
# [INFO] 10.244.x.x - "A IN example.com.cluster.local" NXDOMAIN
# [INFO] 10.244.x.x - "A IN example.com" NOERROR  ← 4번째에서 성공
```

### 3. ndots 설정 비교 실험

```bash
# ndots:5 (기본) 파드
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ndots5-test
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
EOF

# ndots:1 (최적화) 파드
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ndots1-test
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "1"
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
EOF

# 각 파드에서 외부 도메인 조회 시간 비교
time kubectl exec ndots5-test -- wget -q http://example.com -O /dev/null
time kubectl exec ndots1-test -- wget -q http://example.com -O /dev/null
# ndots:1이 더 빠름 (불필요한 NXDOMAIN 쿼리 없음)
```

### 4. StatefulSet Pod의 DNS 확인

```bash
# StatefulSet 생성
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
EOF

# 각 Pod에 개별 DNS 이름 확인
kubectl run dns-test --image=busybox --rm -it -- sh
nslookup web-0.web-headless.default.svc.cluster.local
nslookup web-1.web-headless.default.svc.cluster.local
nslookup web-2.web-headless.default.svc.cluster.local
```

### 5. CoreDNS 메트릭으로 DNS 쿼리 현황 파악

```bash
# CoreDNS Prometheus 메트릭 직접 조회
COREDNS_POD=$(kubectl get pods -n kube-system -l k8s-app=kube-dns -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n kube-system $COREDNS_POD -- \
  wget -qO- http://localhost:9153/metrics | grep -E "coredns_dns_request|coredns_dns_response"

# 주요 메트릭:
# coredns_dns_requests_total            ← 전체 쿼리 수
# coredns_dns_responses_total{rcode="NXDOMAIN"} ← NXDOMAIN 응답 수
# coredns_dns_request_duration_seconds  ← 응답 시간
```

---

## 📊 DNS 이름 형식 정리

| 대상 | 형식 | 예시 |
|-----|-----|-----|
| 같은 NS Service | `<service>` | `my-svc` |
| 다른 NS Service | `<service>.<ns>` | `my-svc.prod` |
| Service FQDN | `<service>.<ns>.svc.cluster.local` | `my-svc.prod.svc.cluster.local` |
| StatefulSet Pod | `<pod>.<service>.<ns>.svc.cluster.local` | `web-0.web-headless.default.svc.cluster.local` |
| Pod IP DNS | `<ip>.<ns>.pod.cluster.local` | `10-244-1-5.default.pod.cluster.local` |
| Headless Service | `<service>.<ns>.svc.cluster.local` | 파드 IP 목록 반환 |

---

## ⚖️ 트레이드오프

**CoreDNS 캐시 TTL 설정**

기본 TTL이 30초이므로 파드 교체 후 최대 30초 동안 이전 ClusterIP가 캐시될 수 있다. TTL을 줄이면 최신성이 높아지지만 CoreDNS 쿼리가 증가한다. Service의 ClusterIP는 Service 삭제 전까지 변하지 않으므로 일반적으로 기본 TTL이 적절하다. 파드 IP가 직접 캐시되는 Headless Service의 경우 파드 교체 후 TTL 동안 연결 오류가 발생할 수 있어 짧은 TTL이 적합하다.

**CoreDNS 고가용성**

CoreDNS가 죽으면 클러스터 내 모든 Service DNS 조회가 실패한다. 기본적으로 replicas=2로 구성된다. 대규모 클러스터에서는 replicas를 더 늘리고, NodeLocal DNSCache(각 노드에 DNS 캐시 에이전트 설치)를 통해 CoreDNS 부하를 분산한다.

---

## 📌 핵심 정리

```
CoreDNS 역할:
  파드 /etc/resolv.conf의 nameserver로 등록 (10.96.0.10)
  Service 이름 → ClusterIP 변환 (A 레코드)
  StatefulSet 파드별 DNS 제공
  외부 도메인은 노드 DNS로 포워딩

DNS 이름 계층:
  my-svc → my-svc.default.svc.cluster.local (자동 완성)
  ndots:5: 점 5개 미만이면 search domain 먼저 시도

ndots:5의 문제:
  외부 도메인 조회 시 3번의 NXDOMAIN 후 실제 쿼리
  → CoreDNS 부하, 지연 발생
  해결: ndots 낮추기, FQDN(끝에 점), autopath 플러그인

CoreDNS 설정:
  kube-system/coredns ConfigMap (Corefile)
  플러그인 체인: kubernetes → cache → forward → ...
  kubectl edit configmap coredns -n kube-system 으로 수정
```

---

## 🤔 생각해볼 문제

**Q1.** 파드를 재시작하면 새로운 IP가 할당되는데, CoreDNS의 캐시 때문에 기존 IP가 반환될 수 있는가?

<details>
<summary>해설 보기</summary>

Headless Service를 통한 파드 직접 IP 조회의 경우 가능하다. Headless Service의 DNS는 파드 IP 목록을 직접 반환하므로, 파드가 재시작되어 IP가 바뀌면 CoreDNS 캐시(TTL 30초)가 만료되기 전까지 이전 IP가 반환될 수 있다. 반면 일반 ClusterIP Service는 파드 IP가 아닌 ClusterIP를 반환하므로, 파드 재시작과 무관하게 동일한 ClusterIP가 반환된다. 대신 kube-proxy의 iptables 규칙이 새 파드 IP를 가리키도록 갱신된다.

</details>

**Q2.** CoreDNS가 응답하지 않을 때 파드는 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

DNS 쿼리 타임아웃이 발생한다. `/etc/resolv.conf`의 `timeout` 옵션(기본 5초)만큼 대기 후 재시도한다. CoreDNS가 완전히 다운된 경우 DNS 기반 Service 이름 해석이 모두 실패한다. 이미 IP를 캐시한 경우 캐시 TTL(30초) 동안은 동작하지만, TTL 만료 후 갱신이 안 되어 연결 실패가 발생한다. 이 때문에 프로덕션에서 CoreDNS를 최소 2개 이상, 다른 노드에 분산 배치해야 한다. NodeLocal DNSCache를 사용하면 각 노드에 캐시가 있어 CoreDNS 장애 영향을 줄일 수 있다.

</details>

**Q3.** 클러스터 내부에서 외부 DNS(예: `internal.mycompany.com`)를 조회해야 한다. CoreDNS를 어떻게 설정하는가?

<details>
<summary>해설 보기</summary>

CoreDNS Corefile에 조건부 포워딩을 추가한다. `kubectl edit configmap coredns -n kube-system`으로 편집해 다음을 추가한다. `mycompany.com:53 { forward . 10.0.0.1 (내부 DNS IP) }`. 이렇게 하면 `mycompany.com` 도메인 쿼리만 내부 DNS 서버로 포워딩하고, 나머지 클러스터 DNS는 기존 kubernetes 플러그인이 처리한다. Corefile 수정 후 `reload` 플러그인이 자동으로 CoreDNS를 재로드한다.

</details>

---

> ⬅️ 이전: [05. Ingress — L7 라우팅과 Nginx Controller](./05-ingress.md)  
> ➡️ 다음: [07. Network Policy — 파드 간 트래픽 제어](./07-network-policy.md)
