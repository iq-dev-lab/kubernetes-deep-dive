# Service와 kube-proxy — iptables 규칙 생성

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ClusterIP는 실제 네트워크 인터페이스에 바인딩되지 않는데 어떻게 동작하는가?
- kube-proxy가 생성하는 iptables 체인은 어떤 구조이고, 패킷이 어떤 경로를 거치는가?
- `iptables-save | grep <service-name>`으로 실제 규칙을 어떻게 읽는가?
- IPVS 모드는 iptables 모드와 어떻게 다르고, 어떤 경우에 유리한가?
- kube-proxy 없이도 Service가 동작할 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"Service를 만들었는데 연결이 안 된다"는 문제의 상당수가 kube-proxy 레벨에서 발생한다. iptables 규칙을 직접 확인하면 ClusterIP 트래픽이 실제 파드 IP로 변환되는 DNAT 규칙이 있는지, 파드 IP가 Endpoints에 등록됐는지 즉시 알 수 있다. 이 계층을 이해하지 못하면 애플리케이션 문제와 인프라 문제를 구분할 수 없다.

> 🔗 **linux-for-backend-deep-dive 연결:** iptables의 체인(PREROUTING/OUTPUT/FORWARD), 타겟(DNAT/SNAT/MASQUERADE)의 원리는 linux-deep-dive에서 다뤘다. 여기서는 kube-proxy가 이 iptables를 어떻게 사용하는지 집중한다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: Service ClusterIP로 접근이 안 됨

원리를 모를 때의 판단:
  "Service가 잘못 만들어졌나?" → kubectl delete service → 재생성
  "파드 문제인가?" → kubectl exec로 파드 직접 접근 → 정상
  → ClusterIP 문제인 것은 알지만 원인 모름

실제 진단 방법:
  # Endpoints가 비어있는가?
  kubectl get endpoints my-service
  → 비어있으면: Selector 레이블 불일치
    kubectl get pod --show-labels vs kubectl describe svc

  # iptables 규칙이 있는가?
  iptables-save | grep my-service
  → 없으면: kube-proxy 이상
    kubectl get pods -n kube-system | grep kube-proxy
    kubectl logs -n kube-system kube-proxy-xxx

  # 규칙은 있는데 안 됨?
  iptables-save | grep KUBE-SEP  → 파드 IP 확인
  → 파드 IP가 오래된 것? Endpoints 갱신 지연
```

---

## ✨ 올바른 접근 (After — kube-proxy를 이해한 진단)

```
Service 동작 확인 3단계:

1. Endpoints 확인 (파드가 등록됐는가?)
   kubectl get endpoints <service>
   → 비어있으면: selector 레이블 불일치

2. iptables DNAT 규칙 확인 (kube-proxy가 규칙 생성했는가?)
   iptables -t nat -L KUBE-SERVICES -n | grep <clusterIP>
   → 없으면: kube-proxy 이상

3. 실제 파드 IP로 직접 접근 (파드 자체 문제인가?)
   kubectl exec test-pod -- curl <pod-ip>:<port>
   → 안 되면: 애플리케이션 문제
```

---

## 🔬 내부 동작 원리

### ClusterIP의 실체 — 가상 IP

ClusterIP는 어떤 실제 네트워크 인터페이스에도 바인딩되지 않는다. `10.96.43.125:80`으로 보내진 패킷은 해당 IP를 가진 장치가 없음에도 동작한다. 어떻게 가능한가?

```
답: iptables의 PREROUTING 체인이 패킷이 라우팅되기 전에 가로채서
    실제 파드 IP로 DNAT(Destination NAT) 변환

패킷 흐름:
  클라이언트 → 10.96.43.125:80
       │
       ▼ iptables PREROUTING (패킷이 라우팅 테이블 결정 전)
  KUBE-SERVICES 체인 확인
  → 10.96.43.125:80 매칭
  → KUBE-SVC-<hash> 체인으로 점프
  → 파드 IP 중 하나로 DNAT (10.244.1.5:8080)
       │
       ▼ DNAT 후 목적지 IP 변경됨
  10.244.1.5:8080 패킷으로 라우팅
  → 실제 파드로 전달
```

### kube-proxy iptables 체인 구조

kube-proxy는 Service와 Endpoints 변화를 Watch하고, 변화가 있을 때마다 iptables 규칙을 업데이트한다.

```
my-service (ClusterIP: 10.96.43.125, Port: 80 → 8080)
파드: pod-a (10.244.1.5), pod-b (10.244.1.6)

생성되는 iptables 체인 (nat 테이블):

PREROUTING
  └─► KUBE-SERVICES

KUBE-SERVICES
  -d 10.96.43.125/32 -p tcp --dport 80 -j KUBE-SVC-XXXXXXXX
  
KUBE-SVC-XXXXXXXX (로드밸런싱: 각 파드로 균등 분배)
  -m statistic --mode random --probability 0.5 -j KUBE-SEP-AAAAAAAA  ← 50% pod-a
  -j KUBE-SEP-BBBBBBBB                                                ← 50% pod-b

KUBE-SEP-AAAAAAAA (pod-a로 DNAT)
  -p tcp -j DNAT --to-destination 10.244.1.5:8080

KUBE-SEP-BBBBBBBB (pod-b로 DNAT)
  -p tcp -j DNAT --to-destination 10.244.1.6:8080

OUTPUT (로컬 프로세스에서 발생하는 패킷도 처리)
  └─► KUBE-SERVICES  (같은 체인 재사용)
```

**확률 기반 로드밸런싱**

```
파드 3개인 경우 확률 설정:
  KUBE-SVC-xxx
    -m statistic --probability 0.333 -j KUBE-SEP-1  ← 33.3%
    -m statistic --probability 0.5   -j KUBE-SEP-2  ← 남은 50%의 50% = 33.3%
    -j KUBE-SEP-3                                    ← 나머지 = 33.3%

처음 규칙이 0.333 확률로 선택되지 않으면 다음 규칙으로
다음 규칙은 남은 패킷의 0.5 확률 → 전체로는 0.333
마지막 규칙은 남은 100%
→ 결과적으로 균등 분배
```

### Endpoints와 kube-proxy의 연동

```
파드 Ready 상태 변화 → Endpoints 업데이트 → kube-proxy iptables 갱신:

  1. 파드의 Readiness Probe 실패
  2. kubelet → API Server: pod.status.conditions[Ready] = False
  3. Endpoint Controller (Controller Manager):
     해당 파드 IP를 Endpoints 오브젝트에서 제거
  4. kube-proxy가 Endpoints Watch 이벤트 수신
  5. 해당 파드 IP를 가리키는 KUBE-SEP-xxx 체인 제거
  6. KUBE-SVC-xxx에서 해당 체인으로의 점프 규칙 제거

결과: Readiness 실패한 파드로는 Service 트래픽이 가지 않음
소요 시간: 1~5초 (Probe 주기 + Watch 전파 + iptables 갱신)
```

### IPVS 모드 — 대규모 클러스터를 위한 대안

kube-proxy는 iptables 모드 외에 IPVS(IP Virtual Server) 모드를 지원한다.

```
iptables 모드의 한계:
  Service 1000개 → KUBE-SVC 1000개 + KUBE-SEP 수천 개 생성
  패킷이 선형 체인 순회 → O(n) 성능
  iptables 갱신(iptables-restore)이 전체 규칙 재적용 → 대규모에서 느림

IPVS 모드:
  커널의 IPVS(LVS)를 사용
  해시 테이블 기반 O(1) 라우팅
  10만 개 Service도 일정한 성능
  다양한 로드밸런싱 알고리즘 지원:
    rr(라운드로빈), lc(최소연결), dh(도착지해시), sh(출발지해시) 등

IPVS 활성화:
  kube-proxy --proxy-mode=ipvs
  또는 ConfigMap:
    kind: KubeProxyConfiguration
    mode: "ipvs"
    ipvs:
      scheduler: "rr"
```

---

## 💻 실전 실험

### 1. Service 생성 후 iptables 규칙 확인

```bash
# Service 생성
kubectl create deployment web --image=nginx --replicas=2
kubectl expose deployment web --port=80 --target-port=80

CLUSTER_IP=$(kubectl get svc web -o jsonpath='{.spec.clusterIP}')
echo "ClusterIP: $CLUSTER_IP"

# Kind 노드에서 iptables 규칙 확인
docker exec -it kind-worker bash

# KUBE-SERVICES 체인에서 해당 Service 확인
iptables -t nat -L KUBE-SERVICES -n | grep $CLUSTER_IP
# KUBE-SVC-XXXXXXXX  tcp  --  0.0.0.0/0  10.96.xxx.xxx  tcp dpt:80

# 해당 SVC 체인으로 이동
SVC_CHAIN=$(iptables -t nat -L KUBE-SERVICES -n | grep $CLUSTER_IP | awk '{print $1}')
iptables -t nat -L $SVC_CHAIN -n
# KUBE-SEP-AAA  ... 0.5 probability
# KUBE-SEP-BBB  ...

# SEP 체인에서 실제 DNAT 확인
iptables -t nat -L KUBE-SEP-AAA -n
# DNAT  tcp  -- 0.0.0.0/0  0.0.0.0/0  tcp to:10.244.x.x:80
```

### 2. iptables-save로 전체 규칙 파일 저장 후 분석

```bash
docker exec -it kind-worker bash

# 전체 nat 테이블 규칙 저장
iptables-save -t nat > /tmp/iptables-nat.txt

# Service 관련 규칙 검색
grep -E "KUBE-SVC|KUBE-SEP" /tmp/iptables-nat.txt | grep $CLUSTER_IP

# Service 개수와 SEP(파드) 개수 확인
grep -c "^-A KUBE-SVC" /tmp/iptables-nat.txt  # Service 수
grep -c "^-A KUBE-SEP" /tmp/iptables-nat.txt  # 파드 엔드포인트 수
```

### 3. Endpoints 변화와 iptables 갱신 관찰

```bash
# 터미널 1: Endpoints Watch
kubectl get endpoints web -w

# 터미널 2: 파드 하나의 Readiness 실패 유발
kubectl exec $(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}') \
  -- nginx -s stop

# Endpoints에서 파드 IP 제거 확인
# 이후 iptables에서 해당 KUBE-SEP 규칙 제거 확인
docker exec -it kind-worker bash
watch "iptables -t nat -L KUBE-SVC-XXXXXXXX -n"
```

### 4. ClusterIP로 직접 curl 테스트

```bash
CLUSTER_IP=$(kubectl get svc web -o jsonpath='{.spec.clusterIP}')

# 클러스터 내부 파드에서 ClusterIP 접근
kubectl run test-curl --image=curlimages/curl --rm -it -- curl $CLUSTER_IP:80
# nginx 응답 → ClusterIP를 통한 로드밸런싱 확인

# 여러 번 요청 시 다른 파드 IP로 분배됨 (pod IP가 응답 헤더에 없으면 로그로 확인)
for i in $(seq 10); do
  kubectl exec test-pod -- curl -s $CLUSTER_IP:80 -o /dev/null -w "%{remote_ip}\n"
done
```

### 5. IPVS 모드 확인 (IPVS 설정된 클러스터)

```bash
# kube-proxy 모드 확인
kubectl get configmap -n kube-system kube-proxy -o yaml | grep mode

# IPVS 가상 서버 목록 확인
ipvsadm -L -n
# IP Virtual Server version 1.2.1 (size=4096)
# Prot LocalAddress:Port  Scheduler Flags
#   TCP  10.96.43.125:80  rr
#     -> 10.244.1.5:80   Masq    1      0          0
#     -> 10.244.1.6:80   Masq    1      0          0
```

---

## 📊 kube-proxy 모드 비교

| 항목 | iptables 모드 | IPVS 모드 |
|-----|-------------|---------|
| 라우팅 알고리즘 | 선형 체인 순회 O(n) | 해시 테이블 O(1) |
| Service 1만개 성능 | 급격한 저하 | 일정 유지 |
| 로드밸런싱 방식 | 확률 기반 랜덤 | rr/lc/dh/sh/sed 선택 가능 |
| 규칙 업데이트 | 전체 규칙 재적용 | 변경분만 갱신 |
| 커널 요구사항 | 기본 지원 | ipvs 커널 모듈 필요 |
| 사용 권장 환경 | 소규모 (< 1000 Service) | 대규모 (> 1000 Service) |

---

## ⚖️ 트레이드오프

**iptables 모드의 한계**

Service 수가 수천 개를 넘으면 iptables 갱신에 수십 초가 걸린다. iptables-restore는 전체 규칙을 원자적으로 교체하기 때문이다. 이 시간 동안 기존 규칙은 유지되지만, 새 Service나 파드는 즉시 반영되지 않는다. 대규모 클러스터에서 IPVS 모드나 eBPF 기반 Cilium kube-proxy 대체를 고려해야 한다.

**kube-proxy 없는 대안**

Cilium을 kube-proxy 대체로 사용하면 iptables 없이 eBPF로 Service 라우팅을 처리한다. `--set kubeProxyReplacement=strict`로 설정하면 kube-proxy DaemonSet 없이 동작한다. 성능이 높고 규칙 수에 무관하지만, Cilium 자체의 복잡성이 증가한다.

---

## 📌 핵심 정리

```
ClusterIP 동작 원리:
  실제 인터페이스 없음 → iptables PREROUTING DNAT으로 구현
  KUBE-SERVICES → KUBE-SVC-xxx → KUBE-SEP-xxx → 실제 파드 IP:PORT

iptables 체인 구조:
  KUBE-SERVICES: Service ClusterIP별 분기
  KUBE-SVC-xxx:  로드밸런싱 (확률 기반)
  KUBE-SEP-xxx:  개별 파드로 DNAT

Endpoints 연동:
  파드 Ready → Endpoints 추가 → KUBE-SEP 체인 추가
  파드 NotReady → Endpoints 제거 → KUBE-SEP 체인 제거

IPVS vs iptables:
  iptables: O(n) 순회, 소규모 적합
  IPVS: O(1) 해시, 대규모 적합
  Cilium eBPF: kube-proxy 대체, 최고 성능
```

---

## 🤔 생각해볼 문제

**Q1.** kube-proxy가 모든 노드에서 동일한 iptables 규칙을 생성하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

클라이언트 파드가 어느 노드에 있든 ClusterIP로 Service에 접근할 수 있어야 하기 때문이다. iptables 규칙은 해당 노드를 통과하는 패킷에만 적용된다. 노드 A의 파드가 ClusterIP로 보낸 패킷은 노드 A의 iptables에서 처리된다. 만약 노드 A에 규칙이 없다면 ClusterIP로 접근이 안 된다. 따라서 kube-proxy DaemonSet은 모든 노드에서 실행되며, 각 노드에 동일한 전체 클러스터 Service 규칙을 유지한다.

</details>

**Q2.** ClusterIP로 접근 시 패킷의 출발지 IP는 어떻게 되는가? (SNAT이 일어나는가?)

<details>
<summary>해설 보기</summary>

같은 노드의 파드에서 ClusterIP로 접근하면 DNAT만 일어나고 SNAT은 일어나지 않는다. 파드 IP가 출발지 IP로 유지된 채 목적지 파드에 전달된다. 따라서 목적지 파드는 실제 클라이언트 파드 IP를 볼 수 있다. 단, NodePort나 다른 노드를 통해 들어오는 트래픽은 SNAT이 적용되어 노드 IP로 변환되는 경우가 있다. `externalTrafficPolicy: Local` 설정으로 SNAT을 방지할 수 있다.

</details>

**Q3.** Service의 파드가 0개(scale to 0)인 상태에서 ClusterIP로 접근하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Endpoints 오브젝트가 비어있으므로 KUBE-SVC-xxx 체인에 KUBE-SEP-xxx로의 점프 규칙이 없다. 패킷이 KUBE-SVC 체인에 진입하지만 아무 규칙도 매칭되지 않아 기본 정책(RETURN 또는 DROP)이 적용된다. 클라이언트 입장에서는 connection refused(TCP RST) 또는 timeout이 발생한다. kube-proxy가 규칙을 보존하는 방식에 따라 정확한 동작이 다를 수 있지만, 일반적으로 아무 파드로도 라우팅되지 않는다.

</details>

---

<div align="center">

**[⬅️ 이전: CNI — Flannel과 Calico 내부 동작](./02-cni-plugins.md)** | **[홈으로 🏠](../README.md)** | **[다음: Service 종류 완전 분해 ➡️](./04-service-types.md)**

</div>
