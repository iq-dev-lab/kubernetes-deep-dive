# Service 종류 완전 분해

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ClusterIP, NodePort, LoadBalancer, ExternalName은 각각 어떻게 구현되는가?
- NodePort가 모든 노드에서 동일하게 동작하는 이유는 무엇인가?
- LoadBalancer Service를 생성하면 클라우드에서 어떤 일이 자동으로 일어나는가?
- Headless Service는 ClusterIP와 어떻게 다르고, 언제 사용하는가?
- ExternalName Service는 어떻게 내부 DNS를 외부 도메인으로 연결하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Service 종류를 잘못 선택하면 불필요한 비용이 발생하거나 보안 취약점이 생긴다. "외부에서 접근하려면 LoadBalancer를 써야 한다"고 알고 있지만, 왜 NodePort가 모든 노드에서 동작하는지, LoadBalancer 내부에 NodePort가 포함되는지 모르면 트러블슈팅이 어렵다. 각 종류의 구현 방식을 알면 올바른 선택과 빠른 진단이 가능하다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 개발 환경에서 Service를 LoadBalancer 타입으로 생성
      kubectl get svc → EXTERNAL-IP가 <pending> 상태로 계속 유지

원리를 모를 때의 판단:
  "로드밸런서가 생성 중인가?" → 계속 대기
  "클러스터 문제인가?" → 재설치 검토

실제 원인:
  LoadBalancer 타입은 클라우드 프로바이더(AWS/GCP/Azure)의
  CCM(Cloud Controller Manager)이 실제 로드밸런서를 프로비저닝해야 함
  로컬/베어메탈 환경에서는 CCM이 없어 영원히 pending

해결:
  개발 환경: NodePort 또는 MetalLB(베어메탈 LB) 사용
  Kind/Minikube: minikube tunnel 또는 port-forward
  프로덕션: AWS ELB, GCP NLB 연동 확인
```

---

## ✨ 올바른 접근 (After — Service 종류를 이해한 선택)

```
Service 종류 선택 기준:

ClusterIP (기본)
  클러스터 내부 간 통신 (마이크로서비스 → 마이크로서비스)
  외부 접근 불필요 시

NodePort
  개발/테스트 외부 접근
  베어메탈 환경에서 단순 외부 노출
  특정 포트(30000-32767)로 노드 IP 직접 접근

LoadBalancer
  프로덕션 클라우드 환경 외부 노출
  각 Service마다 별도 LB 생성 (비용 주의)
  다수 Service → Ingress로 대체 검토

Headless (ClusterIP: None)
  StatefulSet의 파드별 DNS 접근 필요 시
  직접 파드 IP 조회가 필요한 클라이언트

ExternalName
  클러스터 내부에서 외부 DNS를 Service처럼 사용
  DB 엔드포인트를 Service 이름으로 추상화
```

---

## 🔬 내부 동작 원리

### ClusterIP — 기본 내부 통신

ClusterIP는 [03. Service와 kube-proxy](./03-service-kube-proxy.md)에서 상세히 다뤘다. 핵심은 iptables DNAT으로 구현되는 가상 IP다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-backend
spec:
  type: ClusterIP   # 기본값
  selector:
    app: my-backend
  ports:
  - port: 80        # ClusterIP:80으로 접근
    targetPort: 8080 # 파드의 8080으로 전달
```

### NodePort — 노드 IP로 외부 접근

NodePort는 ClusterIP 위에 구축된다. ClusterIP 기능을 그대로 유지하면서 추가로 모든 노드의 특정 포트(30000-32767)에서 트래픽을 받아 파드로 전달한다.

```
NodePort 구조:
  외부 클라이언트 → 192.168.1.10:30080 (노드1 IP)
    또는
  외부 클라이언트 → 192.168.1.11:30080 (노드2 IP)
  (어느 노드로 접근해도 동일하게 동작)
  
          ↓ iptables
  KUBE-NODEPORTS 체인
  → KUBE-SVC-xxx 체인
  → KUBE-SEP-xxx → 파드 IP DNAT

kube-proxy가 모든 노드에 30080 포트 Listen 규칙 생성:
  -A KUBE-NODEPORTS -p tcp --dport 30080 -j KUBE-SVC-xxx

왜 모든 노드에서 동작하는가:
  kube-proxy DaemonSet이 모든 노드에서 실행되고
  각 노드에 동일한 KUBE-NODEPORTS 규칙 생성
  → 어느 노드로 들어와도 전체 파드 중 하나로 DNAT됨
  (요청을 받은 노드에 해당 파드가 없어도 다른 노드로 패킷 전달)
```

```yaml
spec:
  type: NodePort
  ports:
  - port: 80         # ClusterIP로 접근 시 포트
    targetPort: 8080  # 파드 포트
    nodePort: 30080   # 노드 포트 (미지정 시 30000-32767에서 자동 할당)
```

**externalTrafficPolicy 주의사항**

```
기본(Cluster):
  어느 노드로 들어와도 모든 파드로 SNAT+DNAT
  → 출발지 IP가 노드 IP로 변환됨 (실제 클라이언트 IP 손실)
  → 균등 로드밸런싱

Local:
  접근한 노드에 있는 파드에만 전달
  → SNAT 없이 클라이언트 IP 유지
  → 해당 노드에 파드가 없으면 패킷 드롭
  → 파드가 있는 노드로만 트래픽 보내야 함 (LB Health Check 연동)
```

### LoadBalancer — 클라우드 로드밸런서 자동 프로비저닝

LoadBalancer 타입은 NodePort 위에 구축되며, 추가로 클라우드 프로바이더에게 실제 로드밸런서 생성을 요청한다.

```
LoadBalancer 동작 흐름:

  1. kubectl apply -f service-loadbalancer.yaml
  
  2. API Server가 Service 오브젝트 생성
     type: LoadBalancer
  
  3. Cloud Controller Manager(CCM)가 Watch로 감지
     (AWS: aws-cloud-controller-manager)
     (GCP: cloud-controller-manager)
  
  4. CCM이 클라우드 API 호출:
     AWS: ELB(Classic) 또는 NLB(Network) 생성
     GCP: Google Cloud Load Balancer 생성
  
  5. 로드밸런서 생성 완료 후 CCM이 Service 업데이트:
     status.loadBalancer.ingress[0].hostname = "xxx.elb.amazonaws.com"
     (또는 .ip)
  
  6. kubectl get svc:
     EXTERNAL-IP: xxx.elb.amazonaws.com
  
  7. 외부 트래픽:
     DNS → ELB IP → 노드 NodePort → iptables → 파드
```

```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  # AWS NLB 사용 (어노테이션)
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
```

### Headless Service — ClusterIP 없는 Service

`clusterIP: None`으로 설정하면 ClusterIP가 할당되지 않는다. DNS 조회 시 ClusterIP 대신 파드 IP 목록을 직접 반환한다.

```
ClusterIP Service DNS 조회:
  nslookup my-service.default.svc.cluster.local
  → 10.96.43.125 (ClusterIP 1개)

Headless Service DNS 조회:
  nslookup my-headless.default.svc.cluster.local
  → 10.244.1.5 (pod-a IP)
  → 10.244.1.6 (pod-b IP)
  → 10.244.2.7 (pod-c IP)
  (파드 IP 목록 직접 반환 → 클라이언트가 선택)

StatefulSet + Headless:
  my-statefulset-0.my-headless.default.svc.cluster.local → 특정 파드 IP
  my-statefulset-1.my-headless.default.svc.cluster.local → 특정 파드 IP
  → 파드 순서/이름으로 정확한 파드에 접근 가능
```

```yaml
spec:
  clusterIP: None   # Headless 설정
  selector:
    app: my-statefulset
  ports:
  - port: 5432
```

### ExternalName — 외부 서비스를 내부 이름으로

ExternalName은 selector 없이 외부 도메인으로 CNAME을 설정한다. iptables 규칙 없이 DNS CNAME으로만 구현된다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: mydb.rds.amazonaws.com  # 외부 DNS

# 파드에서 사용:
# mysql -h external-db.default.svc.cluster.local
# → CNAME → mydb.rds.amazonaws.com 으로 DNS 해석
```

```
활용 사례:
  개발: internal-db.default.svc.cluster.local → 로컬 DB
  프로덕션: internal-db.default.svc.cluster.local → RDS 엔드포인트
  
  애플리케이션 코드는 항상 internal-db를 사용
  환경마다 ExternalName의 externalName만 변경
  → 코드 변경 없이 환경 전환 가능
```

---

## 💻 실전 실험

### 1. 각 Service 타입 생성 및 확인

```bash
# 테스트용 Deployment
kubectl create deployment svc-test --image=nginx --replicas=2

# ClusterIP
kubectl expose deployment svc-test --name=svc-clusterip --port=80

# NodePort
kubectl expose deployment svc-test --name=svc-nodeport --port=80 --type=NodePort

# Service 목록 확인
kubectl get svc
# NAME           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)
# svc-clusterip  ClusterIP   10.96.x.x     <none>        80/TCP
# svc-nodeport   NodePort    10.96.y.y     <none>        80:3xxxx/TCP

# NodePort 번호 확인
NODEPORT=$(kubectl get svc svc-nodeport -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
echo "NodePort URL: http://$NODE_IP:$NODEPORT"
```

### 2. NodePort — 모든 노드에서 동작 확인

```bash
# Kind 클러스터에서 모든 노드 IP 확인
kubectl get nodes -o wide

# 각 노드에서 NodePort 접근 (어느 노드로든 동일하게 동작)
for node in $(kubectl get nodes -o jsonpath='{.items[*].status.addresses[0].address}'); do
  echo "Node $node:"
  curl -s http://$node:$NODEPORT | head -3
done

# iptables에서 NODEPORTS 체인 확인
docker exec -it kind-worker bash
iptables -t nat -L KUBE-NODEPORTS -n | grep $NODEPORT
```

### 3. Headless Service와 DNS 조회

```bash
# Headless Service 생성
kubectl expose deployment svc-test --name=svc-headless \
  --port=80 --cluster-ip=None

# DNS 조회 비교
kubectl run dns-test --image=busybox --rm -it -- sh

# 일반 ClusterIP Service → IP 1개
nslookup svc-clusterip.default.svc.cluster.local
# Address: 10.96.x.x (ClusterIP)

# Headless Service → 파드 IP 목록
nslookup svc-headless.default.svc.cluster.local
# Address: 10.244.1.5
# Address: 10.244.1.6
```

### 4. ExternalName Service 실습

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: external-dns-test
spec:
  type: ExternalName
  externalName: kubernetes.io
EOF

# 파드에서 외부 도메인을 내부 이름으로 접근
kubectl run ext-test --image=curlimages/curl --rm -it -- \
  curl -L http://external-dns-test.default.svc.cluster.local

# DNS 확인 (CNAME 반환)
kubectl run dns-ext --image=busybox --rm -it -- \
  nslookup external-dns-test.default.svc.cluster.local
# canonical name = kubernetes.io
```

### 5. LoadBalancer - EXTERNAL-IP Pending 상태 분석

```bash
# LoadBalancer Service 생성 (Kind에서는 CCM 없어 pending)
kubectl expose deployment svc-test --name=svc-lb --port=80 --type=LoadBalancer

kubectl get svc svc-lb
# NAME    TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)
# svc-lb  LoadBalancer   10.96.z.z    <pending>     80:3xxxx/TCP

# CCM 관련 이벤트 확인
kubectl describe svc svc-lb | grep -A5 "Events:"
# 이벤트 없음 → CCM이 없어 아무 동작도 안 함

# Kind 환경에서 LoadBalancer 시뮬레이션 (MetalLB 또는 kubectl port-forward)
kubectl port-forward svc/svc-lb 8080:80 &
curl http://localhost:8080
```

---

## 📊 Service 종류별 비교

| 항목 | ClusterIP | NodePort | LoadBalancer | Headless | ExternalName |
|-----|---------|---------|------------|---------|-------------|
| 외부 접근 | ❌ | ✅ (노드IP:포트) | ✅ (LB IP/DNS) | ❌ | ❌ (내부용) |
| ClusterIP | 할당 | 할당 | 할당 | None | None |
| iptables 규칙 | ✅ | ✅ | ✅ | ❌ | ❌ |
| DNS 응답 | ClusterIP | ClusterIP | LB IP | 파드 IP 목록 | CNAME |
| 클라우드 비용 | 없음 | 없음 | LB 비용 발생 | 없음 | 없음 |
| 주요 용도 | 내부 통신 | 개발/테스트 | 프로덕션 외부 | StatefulSet | 외부 DB 추상화 |

---

## ⚖️ 트레이드오프

**LoadBalancer당 LB 1개의 비용 문제**

Service마다 LoadBalancer를 생성하면 각각 클라우드 LB 비용이 발생한다. AWS ALB 기준으로 월 $16~20 이상이다. 10개 Service면 월 $160~200. 대신 Ingress + IngressController 1개로 다수의 HTTP Service를 처리하면 LB 비용을 1개로 줄일 수 있다. 단, HTTP/HTTPS가 아닌 TCP/UDP Service는 Ingress 불가 → LoadBalancer 필요.

**NodePort의 포트 범위 제한**

NodePort는 30000-32767 범위만 사용 가능하다. 서비스가 많아지면 포트 관리가 복잡해진다. 또한 노드 IP가 변경되면(클라우드 재배포 등) 외부에서 접근하는 IP가 바뀐다. 프로덕션에서는 안정적인 IP를 위해 LoadBalancer 또는 Ingress를 사용한다.

---

## 📌 핵심 정리

```
Service 종류 계층:
  ClusterIP = 기반 (내부 가상 IP + iptables DNAT)
  NodePort  = ClusterIP + 노드 포트 추가
  LoadBalancer = NodePort + 클라우드 LB 자동 생성

왜 NodePort가 모든 노드에서 동작하는가:
  kube-proxy DaemonSet → 모든 노드에 동일 iptables 규칙
  어느 노드로 들어와도 전체 파드 중 하나로 DNAT

Headless Service:
  clusterIP: None → ClusterIP 없음
  DNS가 파드 IP 직접 반환 → 클라이언트가 특정 파드 선택 가능
  StatefulSet의 순서 보장 DNS에 필수

ExternalName:
  iptables 규칙 없음, 순수 DNS CNAME
  외부 서비스를 내부 이름으로 추상화 → 환경 전환 용이

LoadBalancer PENDING:
  CCM(Cloud Controller Manager)이 없으면 영원히 pending
  로컬/베어메탈 = MetalLB, Kind = port-forward
```

---

## 🤔 생각해볼 문제

**Q1.** NodePort로 특정 노드에 접근했는데, 해당 노드에 파드가 없다. 요청이 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

`externalTrafficPolicy: Cluster`(기본값)인 경우, 해당 노드의 kube-proxy가 KUBE-NODEPORTS 체인에서 KUBE-SVC 체인으로 연결하고, KUBE-SEP 체인에서 다른 노드에 있는 파드 IP로 DNAT한다. 패킷이 다른 노드의 파드로 전달된다. 이 과정에서 출발지 IP가 현재 노드의 IP로 SNAT된다. `externalTrafficPolicy: Local`이면 로컬 파드가 없을 때 패킷이 드롭된다.

</details>

**Q2.** Service를 삭제했을 때 DNS 캐시 때문에 기존 ClusterIP로 접근이 계속 될 수 있는가?

<details>
<summary>해설 보기</summary>

DNS 캐시 TTL 동안은 캐시된 ClusterIP를 사용할 수 있지만, iptables 규칙은 Service 삭제와 함께 제거된다. 따라서 ClusterIP로 패킷을 보내도 iptables에 매칭 규칙이 없어 패킷이 처리되지 않는다. DNS TTL은 CoreDNS 설정에 따라 다르지만 기본 30초다. TTL이 만료되면 DNS 조회 결과도 없어져 연결이 완전히 차단된다.

</details>

**Q3.** 같은 네임스페이스에 ClusterIP Service와 같은 이름의 ExternalName Service를 만들 수 있는가?

<details>
<summary>해설 보기</summary>

불가능하다. 같은 네임스페이스에 같은 이름의 Service는 하나만 존재할 수 있다. Service 오브젝트는 이름으로 식별되며, API Server가 중복 이름의 오브젝트 생성을 거부한다. 환경(개발/프로덕션)별로 ExternalName을 바꾸려면 네임스페이스를 분리하거나, Helm 같은 배포 도구로 조건부로 다른 Service 타입을 배포해야 한다.

</details>

---

> ⬅️ 이전: [03. Service와 kube-proxy — iptables 규칙 생성](./03-service-kube-proxy.md)  
> ➡️ 다음: [05. Ingress — L7 라우팅과 Nginx Controller](./05-ingress.md)
