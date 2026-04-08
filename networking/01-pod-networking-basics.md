# 파드 네트워킹 기초 — IP-per-Pod 모델

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 쿠버네티스가 "모든 파드는 고유한 IP를 가진다"고 보장하는 원리는?
- 파드 간 통신이 NAT 없이 이루어진다는 것이 구체적으로 무슨 의미인가?
- `pause` 컨테이너가 네트워크 네임스페이스를 초기화하고 유지하는 방법은?
- 파드 내 컨테이너들이 같은 Network namespace를 공유하면 어떤 제약이 생기는가?
- 다른 노드의 파드와 통신할 때 패킷이 어떤 경로를 거치는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"파드 IP로 직접 통신할 수 있다"는 사실을 알고 있어도, 그게 왜 가능한지 모르면 네트워크 문제를 진단할 수 없다. CNI 플러그인이 설정한 veth pair, 노드의 라우팅 테이블, VXLAN 터널 — 이 중 하나라도 잘못되면 파드 간 통신이 안 된다. 어디서 끊겼는지 추적하려면 기반 모델을 이해해야 한다.

> 🔗 **linux-for-backend-deep-dive 연결:** Linux network namespace와 veth pair의 원리는 linux-deep-dive에서 다뤘다. 여기서는 그 개념이 쿠버네티스 파드 네트워킹에서 어떻게 활용되는지 집중한다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 파드 A에서 파드 B의 IP로 통신이 안 됨

원리를 모를 때의 판단:
  "같은 클러스터니까 당연히 되야 하는데"
  → 방화벽 팀에 문의 → 클라우드 보안그룹 확인
  → 해결 안 됨

실제 원인 파악 순서:
  1. 같은 노드인가, 다른 노드인가?
     → 다른 노드: CNI 플러그인의 노드 간 라우팅 설정 확인
  
  2. CNI 플러그인이 정상 동작하는가?
     kubectl get pods -n kube-system | grep cni  → Running?
  
  3. 노드 라우팅 테이블 확인
     ip route show | grep 10.244  → 대상 파드 IP 서브넷 경로 있는가?
  
  4. Network Policy가 차단하는가?
     kubectl get networkpolicy -A
  
  5. veth pair 확인
     ip link show | grep veth
     → 파드의 veth가 bridge에 연결됐는가?
```

---

## ✨ 올바른 접근 (After — 네트워킹 모델을 알고 난 진단)

```
쿠버네티스 네트워킹 4가지 규칙 (Kubernetes Networking Model):
  1. 파드 내 컨테이너들은 localhost로 통신 (Network namespace 공유)
  2. 파드 간 NAT 없이 직접 통신 (파드 IP를 그대로 사용)
  3. 노드와 파드 간 NAT 없이 통신
  4. 파드가 인식하는 자신의 IP = 다른 파드가 보는 IP (NAT에 의한 IP 변환 없음)

이 규칙을 구현하는 것이 CNI의 책임
CNI 플러그인이 없거나 잘못 설정되면 2~4번 규칙이 깨짐
```

---

## 🔬 내부 동작 원리

### pause 컨테이너 — 네트워크 네임스페이스의 소유자

파드가 생성될 때 가장 먼저 시작되는 것이 `pause` 컨테이너(infra 컨테이너)다.

```
pause 컨테이너 역할:
  1. 새로운 Linux Network namespace 생성
     (독립적인 네트워크 인터페이스, 라우팅 테이블, iptables 규칙)
  
  2. pause(2) 시스템콜로 무한 대기
     (프로세스가 종료되지 않아야 namespace가 유지됨)
  
  3. 메인 컨테이너들이 이 namespace에 join
     runc config.json의 "namespaces": [{"type": "network", "path": "/proc/<pause-pid>/ns/net"}]

결과:
  pause가 죽으면 namespace가 사라지고 파드 내 모든 컨테이너 재시작
  메인 컨테이너가 재시작되어도 namespace(= IP)는 pause가 유지
```

```bash
# pause 컨테이너 확인
crictl ps | grep pause

# pause 컨테이너의 PID 확인
PAUSE_ID=$(crictl ps | grep pause | grep <pod-name> | awk '{print $1}')
PAUSE_PID=$(crictl inspect $PAUSE_ID | jq '.info.pid')

# pause가 보유한 network namespace 확인
ls -la /proc/$PAUSE_PID/ns/net
# → /proc/<pid>/ns/net → net:[4026531992]  ← namespace inode 번호

# 다른 컨테이너가 같은 namespace를 공유하는지 확인
APP_PID=$(crictl inspect <app-container-id> | jq '.info.pid')
ls -la /proc/$APP_PID/ns/net
# → 같은 net:[4026531992]  ← inode 번호 동일 = 같은 namespace
```

### veth pair와 브리지 — 파드와 노드 네트워크 연결

파드에 IP가 할당되고 노드와 통신하는 방법이다.

```
노드 네트워크 구조:

  파드 A (10.244.1.5)        파드 B (10.244.1.6)
  ┌──────────────────┐       ┌──────────────────┐
  │  eth0 (10.244.1.5)│      │  eth0 (10.244.1.6)│
  └────────┬─────────┘       └────────┬─────────┘
           │ veth pair                 │ veth pair
           │ (가상 이더넷 케이블)      │
  ─────────┴──────────────────────────┴─────────
                 cni0 (Linux 브리지)
                 (10.244.1.1)
  ────────────────────────────────────────────
                    eth0 (노드 물리 인터페이스)
                    (192.168.1.10)

veth pair 동작:
  - 한쪽 끝은 파드의 eth0
  - 다른쪽 끝은 노드의 cni0 브리지에 연결
  - 파드에서 나간 패킷 → veth → 브리지 → 노드 네트워크
```

```bash
# 노드에서 veth pair 확인
ip link show type veth
# 9: veth1234@if3: ...  ← 파드 A의 veth
# 11: veth5678@if3: ... ← 파드 B의 veth

# 브리지 연결 확인
brctl show cni0
# bridge name  bridge id         interfaces
# cni0         8000.xxx          veth1234
#                                veth5678

# 파드 IP와 veth 매핑
ip neigh show dev cni0
# 10.244.1.5 dev cni0 lladdr ... REACHABLE
# 10.244.1.6 dev cni0 lladdr ... REACHABLE
```

### 같은 노드 내 파드 간 통신

```
파드 A (10.244.1.5) → 파드 B (10.244.1.6) 통신:

  1. 파드 A의 eth0에서 패킷 출발
     dst: 10.244.1.6
  
  2. 파드 A의 기본 라우팅 테이블:
     10.244.1.0/24 via 10.244.1.1 dev eth0
     → cni0 브리지(10.244.1.1)로 전송
  
  3. 노드의 cni0 브리지:
     ARP 테이블에서 10.244.1.6의 MAC 주소 확인
     → 해당 veth pair(파드 B)로 전달
  
  4. 파드 B의 eth0 수신
  
  NAT 없음: 출발지 IP(10.244.1.5) 그대로 유지
```

### 다른 노드의 파드 간 통신

이 부분은 CNI 플러그인마다 구현 방식이 다르다. [02. CNI 플러그인](./02-cni-plugins.md)에서 상세히 다룬다. 기본 원리는 다음과 같다.

```
파드 A (노드1: 10.244.1.5) → 파드 C (노드2: 10.244.2.7) 통신:

  1. 파드 A에서 패킷 출발 (dst: 10.244.2.7)

  2. 노드1의 라우팅 테이블:
     10.244.2.0/24 via 192.168.1.11 (노드2 IP)
     → 노드2로 패킷 전송

  3. CNI 플러그인에 따라:
     Flannel(VXLAN): 패킷을 UDP로 캡슐화 → 노드2에서 언캡슐화
     Calico(BGP):    BGP로 라우팅 경로 교환 → 직접 라우팅

  4. 노드2에서 수신 → cni0 브리지 → 파드 C의 veth → 파드 C 수신

  NAT 없음: 10.244.1.5 → 10.244.2.7 그대로 유지
```

### 파드 내 포트 충돌 문제

파드 내 컨테이너들이 Network namespace를 공유하므로, 포트 공간도 공유된다.

```
메인 컨테이너: 8080 포트 listen
Sidecar:       8080 포트 listen 시도 → 포트 충돌! 시작 실패

올바른 포트 배분:
  메인 앱:    8080 (HTTP)
  Envoy:     15001 (아웃바운드 인터셉트)
             15006 (인바운드 인터셉트)
             15090 (Prometheus 메트릭)
  Datadog:   8125 (StatsD)

컨테이너 간 포트 번호를 레이블이나 문서로 명시적 관리 필요
```

---

## 💻 실전 실험

### 1. 파드 IP 구조 확인

```bash
# 파드의 IP 확인
kubectl get pod -o wide
# NAME    READY   STATUS    IP            NODE
# pod-a   1/1     Running   10.244.1.5    worker-1
# pod-b   1/1     Running   10.244.1.6    worker-1
# pod-c   1/1     Running   10.244.2.7    worker-2

# 파드 네트워크 인터페이스 직접 확인
kubectl exec pod-a -- ip addr show eth0
# 3: eth0@if9: <BROADCAST,MULTICAST,UP>
#     inet 10.244.1.5/24 brd 10.244.1.255 scope global eth0
```

### 2. NAT 없는 직접 통신 확인

```bash
# 파드 A에서 파드 C (다른 노드)로 직접 통신
kubectl exec pod-a -- curl 10.244.2.7:80

# 파드 C에서 수신된 요청의 출발지 IP 확인
kubectl exec pod-c -- sh -c 'cat /proc/net/nf_conntrack | grep 10.244.1.5'
# 출발지가 10.244.1.5 그대로 (NAT 변환 없음)
```

### 3. pause 컨테이너와 Network namespace 공유 확인

```bash
docker exec -it kind-worker bash

# 실행 중인 모든 컨테이너의 network namespace 목록
for pid in $(ls /proc | grep -E '^[0-9]+$'); do
  ns=$(ls -la /proc/$pid/ns/net 2>/dev/null | awk '{print $NF}')
  cmd=$(cat /proc/$pid/cmdline 2>/dev/null | tr '\0' ' ' | cut -c1-30)
  echo "$ns $pid $cmd"
done 2>/dev/null | grep -E "pause|nginx" | sort | head -20

# 같은 namespace(inode)를 공유하는 pause와 nginx 확인
```

### 4. 노드 라우팅 테이블 확인

```bash
docker exec -it kind-worker bash

# 파드 CIDR에 대한 라우팅 경로
ip route show | grep 10.244
# 10.244.0.0/24 via 192.168.xxx.xxx dev eth0    ← 다른 노드의 파드 CIDR
# 10.244.1.0/24 dev cni0 proto kernel scope link ← 현재 노드의 파드 CIDR
# 10.244.1.5 dev vethxxxx scope link            ← 개별 파드로의 경로
```

### 5. 다른 노드 파드 통신 경로 추적

```bash
# traceroute로 패킷 경로 추적
kubectl run tracert --image=busybox --rm -it -- traceroute 10.244.2.7
# 1. 10.244.1.1 (cni0 브리지)
# 2. 192.168.1.11 (노드2, VXLAN을 사용하면 한 hop)
# 3. 10.244.2.7 (대상 파드)
```

---

## 📊 파드 네트워킹 계층 구조

| 계층 | 구성 요소 | 역할 | 확인 명령어 |
|-----|---------|-----|-----------|
| 파드 내부 | Network namespace, eth0 | 컨테이너 간 공유 | `ip addr` (파드 내) |
| 파드-노드 | veth pair | 파드↔노드 연결 | `ip link show type veth` |
| 노드 내 | cni0 브리지 | 같은 노드 파드 간 | `brctl show cni0` |
| 노드 간 | CNI 플러그인 (VXLAN/BGP) | 다른 노드 파드 통신 | `ip route` |
| 정책 | Network Policy | 트래픽 필터링 | `kubectl get networkpolicy` |

---

## ⚖️ 트레이드오프

**IP-per-Pod 모델의 장점**

NAT 없이 파드 IP로 직접 통신하므로 디버깅이 단순하다. `tcpdump`, `curl`로 실제 파드 IP를 사용할 수 있다. 포트 충돌 걱정 없이 모든 파드가 원하는 포트를 사용할 수 있다.

**IP-per-Pod 모델의 단점**

파드 수만큼 IP가 필요하다. 클러스터 CIDR을 충분히 크게 설정해야 한다. 기본 `/16` 서브넷이면 65534개 파드 IP를 제공하지만, 이를 초과하면 파드 생성이 실패한다. 또한 파드 IP는 파드 재시작 시 변경되므로, 파드 IP를 직접 사용하는 것은 권장하지 않는다. Service를 통해 안정적인 엔드포인트를 사용해야 한다.

---

## 📌 핵심 정리

```
IP-per-Pod 모델의 핵심:
  모든 파드는 고유한 IP를 가짐
  파드 간 NAT 없이 직접 통신 (IP 그대로 사용)
  파드 내 컨테이너들은 Network namespace 공유 = 같은 IP + localhost 통신

pause 컨테이너:
  파드의 Network namespace를 생성하고 소유
  메인 컨테이너들이 pause의 namespace에 join
  메인 컨테이너 재시작해도 pause가 namespace 유지 → IP 유지

veth pair:
  파드와 노드 브리지(cni0)를 연결하는 가상 케이블
  한쪽: 파드 eth0, 다른 쪽: cni0 브리지
  CNI 플러그인이 파드 생성 시 자동 설정

파드 내 포트 공유:
  컨테이너들이 같은 포트 공간 사용
  → 포트 충돌 주의 (같은 파드 내 포트 중복 불가)
```

---

## 🤔 생각해볼 문제

**Q1.** 파드를 삭제하고 재생성하면 IP가 바뀌는가? 바뀐다면 어떻게 안정적인 엔드포인트를 제공하는가?

<details>
<summary>해설 보기</summary>

파드를 삭제하고 재생성하면 IP가 바뀐다. CNI 플러그인이 새 파드에 새로운 IP를 할당하기 때문이다. 안정적인 엔드포인트는 Service를 통해 제공된다. Service는 ClusterIP라는 고정 가상 IP를 가지며, 레이블 셀렉터로 파드를 선택한다. 파드가 재생성되어 IP가 바뀌어도 Service는 새 파드 IP를 자동으로 반영한다. 클라이언트는 변하지 않는 Service ClusterIP(또는 DNS 이름)로 통신한다.

</details>

**Q2.** 파드가 Running 상태인데 파드 IP로 ping이 안 된다. 무엇을 확인해야 하는가?

<details>
<summary>해설 보기</summary>

순서대로 확인한다. (1) 같은 노드인지 다른 노드인지 확인. (2) CNI 파드가 Running인지 확인(`kubectl get pods -n kube-system | grep cni`). (3) 노드의 라우팅 테이블에 파드 CIDR 경로가 있는지 확인(`ip route show`). (4) veth pair가 브리지에 연결됐는지 확인(`brctl show cni0`). (5) Network Policy가 ICMP를 차단하는지 확인. (6) 보안 그룹/방화벽이 파드 CIDR 트래픽을 허용하는지 확인(클라우드 환경).

</details>

**Q3.** 클러스터 파드 CIDR를 `/16`으로 설정했는데, 실제로 65534개 파드를 모두 생성할 수 있는가?

<details>
<summary>해설 보기</summary>

이론적으로는 가능하지만 실제로는 제약이 있다. 노드당 파드 수 한계(기본 110개), 노드 수 한계, etcd 성능 한계가 있다. 또한 `/16`을 여러 노드에 서브넷으로 나눠 배분하는데(예: 노드당 `/24` = 254개 IP), 노드가 1000개면 1000 × 254 = 254,000개 IP가 필요하므로 `/16`(65534개)으로는 부족할 수 있다. 대규모 클러스터는 `/12`나 더 큰 CIDR을 사용한다. kubeadm의 기본 `--pod-network-cidr=10.244.0.0/16`은 소규모~중규모 클러스터용이다.

</details>

---

<div align="center">

**[⬅️ 이전: Chapter 2 — Init Container와 Sidecar 패턴](../pod-lifecycle/05-init-container-sidecar.md)** | **[홈으로 🏠](../README.md)** | **[다음: CNI — Flannel과 Calico 내부 동작 ➡️](./02-cni-plugins.md)**

</div>
