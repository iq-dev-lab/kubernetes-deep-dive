# Network Policy — 파드 간 트래픽 제어

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Network Policy가 없을 때 기본 트래픽 허용 정책은 무엇인가?
- Default Deny 정책은 어떻게 구현하고, 이후 필요한 트래픽만 어떻게 허용하는가?
- Network Policy가 iptables 또는 eBPF 레벨에서 어떻게 구현되는가?
- `podSelector`, `namespaceSelector`, `ipBlock`의 차이는 무엇인가?
- Calico와 Cilium은 Network Policy 구현 방식이 어떻게 다른가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

기본적으로 쿠버네티스 클러스터 내 모든 파드는 서로 자유롭게 통신할 수 있다. DB 파드가 어느 파드에서든 접근 가능하다는 의미다. 보안 침해가 발생해 하나의 파드가 탈취되면, 탈취된 파드에서 클러스터 내 모든 파드에 접근할 수 있다. Network Policy는 이 "평평한 네트워크"에 방화벽 규칙을 추가해 최소 권한 원칙을 적용한다.

금융권, 의료 등 규제 환경에서는 네트워크 격리가 컴플라이언스 요구사항이기도 하다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: Network Policy를 적용했는데 허용해야 할 트래픽도 차단됨

원리를 모를 때의 판단:
  "Policy가 잘못 설정됐나?" → 레이블 확인 → 맞아 보임
  "CNI가 지원하지 않는 건가?" → 검색
  → 원인 모름

두 가지 흔한 실수:

실수 1: Flannel 단독 클러스터에서 Network Policy 사용
  Flannel은 Network Policy를 구현하지 않음
  Policy를 만들어도 실제로 차단이 안 됨 (무시됨)
  → Calico 또는 Cilium 필요

실수 2: ingress/egress 방향 혼동
  ingress:  수신 트래픽 (들어오는 것)
  egress:   발신 트래픽 (나가는 것)
  
  DB 파드에 ingress policy만 설정 → 앱 파드에서 DB로의 수신 제어 OK
  그러나 앱 파드의 egress policy가 없으면
  앱 파드 → 다른 외부 목적지로 나가는 트래픽은 허용됨
  완전한 최소 권한 = ingress + egress 양쪽 설정
```

---

## ✨ 올바른 접근 (After — Network Policy를 이해한 설계)

```
권장 패턴: Default Deny → 필요한 것만 허용

1단계: 네임스페이스에 Default Deny 적용
  모든 인바운드/아웃바운드 차단

2단계: 필요한 통신만 명시적으로 허용
  앱 → DB: 앱 파드 → DB 파드 5432 허용
  인터넷 → 앱: Ingress Controller → 앱 파드 8080 허용
  앱 → CoreDNS: 앱 파드 → kube-system DNS 53 허용
  앱 → 외부 API: 앱 파드 → 특정 IP 범위 443 허용

트래픽 매트릭스로 먼저 설계:
  From \ To    | frontend | backend | db  | coredns | external
  -------------|----------|---------|-----|---------|----------
  internet     |    ✅    |   ❌    | ❌  |   ❌    |   -
  frontend     |    -     |   ✅    | ❌  |   ✅    |   ✅
  backend      |    ❌    |    -    | ✅  |   ✅    |   ✅
  db           |    ❌    |   ❌    |  -  |   ✅    |   ❌
```

---

## 🔬 내부 동작 원리

### Network Policy가 없을 때 — 기본 허용

```
Network Policy 미설정 클러스터:
  모든 파드 간 통신 허용 (방향 무관)
  모든 네임스페이스 간 통신 허용
  외부 → 파드 통신도 기본 허용

이 상태에서 탈취된 파드의 가능한 행동:
  ├─ DB 파드 (5432)에 직접 연결해 데이터 덤프
  ├─ 다른 마이크로서비스 API 호출
  ├─ kube-dns 조회로 Service 목록 파악
  ├─ kubelet API (10250)에 접근 시도
  └─ 외부 C2 서버로 데이터 유출
```

### Default Deny 정책 구현

```yaml
# 네임스페이스 내 모든 인바운드 차단
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}      # 빈 selector = 네임스페이스 내 모든 파드
  policyTypes:
  - Ingress             # Ingress만 정의하고 규칙 없음 = 모든 인바운드 차단
---
# 모든 아웃바운드도 차단
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress              # Egress만 정의하고 규칙 없음 = 모든 아웃바운드 차단
```

### 필요한 트래픽 허용

```yaml
# DB에 대한 ingress 허용 (백엔드 파드에서만)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres          # 이 policy가 적용되는 파드 (수신자)
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend        # 이 파드에서 오는 트래픽만 허용
      namespaceSelector:      # AND 조건: 같은 namespace도 명시
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 5432
---
# 백엔드가 DB에 접근하는 egress 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-egress-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend             # 이 policy가 적용되는 파드 (발신자)
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres        # DB 파드로만 나가는 트래픽 허용
    ports:
    - protocol: TCP
      port: 5432
  # DNS도 허용 (없으면 hostname 해석 불가)
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### podSelector + namespaceSelector AND/OR 조건

```yaml
# OR 조건 (from 배열에 각각 독립 항목)
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend       # frontend 파드 OR
  - namespaceSelector:
      matchLabels:
        env: staging        # staging 네임스페이스의 어떤 파드
  → frontend 파드 또는 staging NS의 파드 중 하나라도 해당하면 허용

# AND 조건 (from 배열 한 항목에 둘 다)
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend       # frontend 파드이면서 AND
    namespaceSelector:
      matchLabels:
        env: production     # production 네임스페이스인 파드만 허용
  → 두 조건 모두 만족해야 허용

중요: 위의 - 위치가 AND/OR를 결정
  - 한 항목 내 podSelector + namespaceSelector → AND
  - 배열의 별도 항목 → OR
```

### ipBlock — 외부 IP 범위 제어

```yaml
# 특정 외부 IP 범위에서만 접근 허용
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8         # 사내 IP 대역 허용
      except:
      - 10.1.1.0/24            # 이 서브넷 제외

# 외부 SaaS API로의 egress 허용
egress:
- to:
  - ipBlock:
      cidr: 203.0.113.0/24    # 특정 외부 서비스 IP 대역
  ports:
  - protocol: TCP
    port: 443
```

### CNI별 구현 방식

**Calico — iptables / eBPF**

```
Calico의 Felix 에이전트가 각 노드에서 Network Policy를 iptables 규칙으로 변환:

  Network Policy → Felix → iptables
  
  예: allow-backend-to-db 정책
  iptables -A FORWARD -s 10.244.1.5 -d 10.244.2.7 -p tcp --dport 5432 -j ACCEPT
  iptables -A FORWARD -d 10.244.2.7 -p tcp --dport 5432 -j DROP

Calico eBPF 모드 (Calico 3.13+):
  XDP 레벨에서 패킷 처리 → iptables보다 성능 높음
  대규모 정책에서도 선형적 성능 유지
```

**Cilium — eBPF 전용**

```
Cilium은 iptables 사용 없이 eBPF만으로 모든 정책 구현:

  Network Policy → Cilium Agent → BPF 맵
  
  패킷이 XDP/TC 레벨에서 BPF 프로그램으로 처리:
    BPF 맵 조회(O(1)) → 허용/차단 결정
    
  장점:
    정책 수에 무관한 O(1) 조회 성능
    L7 정책 지원 (HTTP path, gRPC method 레벨 제어)
    - 예: GET /api/public 허용, POST /api/admin 차단

Cilium L7 Network Policy:
  apiVersion: cilium.io/v2
  kind: CiliumNetworkPolicy
  spec:
    endpointSelector:
      matchLabels:
        app: api
    ingress:
    - toPorts:
      - ports:
        - port: "80"
        rules:
          http:
          - method: GET
            path: /api/public     # GET /api/public만 허용
```

---

## 💻 실전 실험

### 1. Default Deny 적용 및 통신 차단 확인

```bash
# 테스트용 파드 2개 생성
kubectl run client --image=curlimages/curl -- sleep 3600
kubectl run server --image=nginx
kubectl expose pod server --port=80

SERVER_IP=$(kubectl get pod server -o jsonpath='{.status.podIP}')

# Policy 없을 때: 통신 가능
kubectl exec client -- curl -s http://$SERVER_IP:80 | head -1
# <!DOCTYPE html>   ← 성공

# Default Deny Ingress 적용
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# Policy 적용 후: 통신 차단
kubectl exec client -- curl --max-time 5 http://$SERVER_IP:80
# curl: (28) Connection timed out  ← 차단됨
```

### 2. 특정 파드에서만 허용

```bash
# 허용 Policy 추가 (client → server만 허용)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-to-server
spec:
  podSelector:
    matchLabels:
      run: server         # server 파드에 적용
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: client     # client 파드에서 오는 트래픽만 허용
    ports:
    - port: 80
EOF

# client → server: 허용
kubectl exec client -- curl -s http://$SERVER_IP:80 | head -1
# <!DOCTYPE html>   ← 성공

# 다른 파드 → server: 차단 확인
kubectl run other --image=curlimages/curl -- sleep 3600
kubectl exec other -- curl --max-time 5 http://$SERVER_IP:80
# 타임아웃 (차단됨)
```

### 3. 네임스페이스 간 트래픽 제어

```bash
# 별도 네임스페이스 생성
kubectl create namespace external-ns
kubectl label namespace external-ns env=external

# external-ns의 파드 → default의 server: 차단 확인
kubectl run ext-client -n external-ns --image=curlimages/curl -- sleep 3600
kubectl exec -n external-ns ext-client -- curl --max-time 5 http://$SERVER_IP:80
# 타임아웃 (네임스페이스 간 차단)

# 특정 네임스페이스에서만 허용하는 Policy 추가
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-ns
spec:
  podSelector:
    matchLabels:
      run: server
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: external
EOF

kubectl exec -n external-ns ext-client -- curl -s http://$SERVER_IP:80 | head -1
# <!DOCTYPE html>   ← 허용됨
```

### 4. DNS egress 허용 패턴 (실무 필수)

```bash
# Egress Default Deny + DNS + 특정 Service 허용
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  # DNS 허용 (이게 없으면 hostname으로 아무것도 못 함)
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # DB 접근 허용
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - port: 5432
EOF
```

### 5. 현재 적용된 Network Policy 확인 및 디버깅

```bash
# 적용된 Policy 목록
kubectl get networkpolicy -A

# 특정 파드에 어떤 Policy가 적용됐는지 확인
kubectl describe pod server | grep -A3 "Labels:"
kubectl get networkpolicy -o yaml | grep -A5 "podSelector"

# Calico 환경: calico 진단 도구
# kubectl exec -n calico-system <calico-pod> -- calicoctl get policy
```

---

## 📊 Network Policy 구현체 비교

| 항목 | Calico | Cilium | Weave Net |
|-----|--------|--------|---------|
| L3/L4 Network Policy | ✅ | ✅ | ✅ |
| L7 HTTP 정책 | ❌ | ✅ CiliumNetworkPolicy | ❌ |
| 구현 방식 | iptables 또는 eBPF | eBPF 전용 | iptables |
| 대규모 성능 | iptables: O(n), eBPF: O(1) | O(1) | O(n) |
| 정책 수 영향 | iptables 모드에서 저하 | 없음 | 저하 |
| 진단 도구 | calicoctl | Hubble UI | 없음 |

---

## ⚖️ 트레이드오프

**Default Deny의 운영 복잡성**

Default Deny를 적용하면 새 서비스를 배포할 때마다 Network Policy를 함께 작성해야 한다. 개발팀이 이를 잊으면 서비스가 동작하지 않는다. Policy를 템플릿으로 만들고, CI/CD 파이프라인에서 Network Policy 파일 존재를 검증하는 단계를 추가하는 것이 권장된다.

**Flannel + Calico (Canal) 구성**

Flannel만으로는 Network Policy를 지원하지 않으므로, Flannel(패킷 라우팅) + Calico(정책 엔진)를 함께 사용하는 Canal 구성을 사용할 수 있다. 단순히 Calico를 독립 CNI로 사용하는 것보다 구성이 복잡하므로 신규 클러스터라면 Calico 단독 또는 Cilium을 권장한다.

---

## 📌 핵심 정리

```
기본 상태:
  Network Policy 없음 = 모든 파드 간 통신 허용
  → 탈취된 파드가 자유롭게 클러스터 탐색 가능

보안 설계 패턴:
  Default Deny (podSelector: {}) 먼저 적용
  → 필요한 통신만 명시적 허용
  → Ingress(수신) + Egress(발신) 양쪽 고려
  → DNS(UDP 53) egress 반드시 허용

AND vs OR 조건:
  같은 from 항목 내 podSelector + namespaceSelector → AND
  배열의 별도 항목 → OR

구현체 선택:
  Flannel: Network Policy 미지원
  Calico: iptables 또는 eBPF, L3/L4
  Cilium: eBPF 전용, L7 HTTP 정책까지 지원
```

---

## 🤔 생각해볼 문제

**Q1.** Network Policy로 특정 파드의 외부 인터넷 접근을 차단하면서 클러스터 내부 Service에는 접근 가능하게 하려면 어떻게 설정하는가?

<details>
<summary>해설 보기</summary>

Egress 정책에서 클러스터 내부 CIDR만 허용하고 외부를 차단한다. 파드 CIDR(10.244.0.0/16)과 Service CIDR(10.96.0.0/12)을 ipBlock으로 허용하고, DNS(UDP 53)도 허용한다. 외부 인터넷(0.0.0.0/0)에 대한 egress 허용 규칙은 추가하지 않는다. 단, Service를 IP가 아닌 ClusterIP로 접근하므로 10.96.0.0/12 범위를 허용하면 된다. Default Deny Egress가 적용된 상태에서 명시적으로 허용한 범위 외에는 모두 차단된다.

</details>

**Q2.** Network Policy는 이미 수립된(established) TCP 연결에도 즉시 적용되는가?

<details>
<summary>해설 보기</summary>

CNI 구현체에 따라 다르다. Calico iptables 모드에서는 기존 연결 추적(conntrack) 엔트리가 유지되어 이미 수립된 연결에는 새 정책이 즉시 적용되지 않을 수 있다. Cilium eBPF 모드는 패킷 레벨에서 처리하므로 기존 연결도 즉시 차단할 수 있다. 실무에서는 Policy 변경 후 기존 연결이 완전히 차단되길 원한다면 관련 파드를 재시작해 기존 연결을 끊는 것이 안전하다.

</details>

**Q3.** 특정 파드가 클러스터 내 어느 파드로든 접근 가능해야 하는 경우(예: 모니터링 에이전트) Network Policy를 어떻게 설정하는가?

<details>
<summary>해설 보기</summary>

모니터링 에이전트 파드에 `egress: - to: []` (모든 대상 허용)를 설정하거나, 다른 모든 파드에 `ingress: - from: - podSelector: matchLabels: app: monitoring` 규칙을 추가한다. 전자가 에이전트 파드에만 설정하므로 관리가 단순하다. 단, Default Deny 정책이 있는 경우 에이전트 파드에 대한 egress 허용 규칙도 있어야 한다. Cilium이라면 CiliumNetworkPolicy의 `endpointSelector`에 에이전트 레이블을 지정하고 `egress: - toEntities: - "all"`로 모든 목적지를 허용하는 것이 명확하다.

</details>

---

<div align="center">

**[⬅️ 이전: CoreDNS — Service 이름 해석](./06-coredns.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — Volume과 파드 ➡️](../storage/01-volumes.md)**

</div>
