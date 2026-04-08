# CNI — Flannel과 Calico 내부 동작

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CNI 플러그인은 kubelet과 어떻게 통신하고, 파드 IP를 어떻게 할당하는가?
- Flannel의 VXLAN 오버레이는 패킷을 어떻게 캡슐화하고, 왜 오버헤드가 발생하는가?
- Calico의 BGP 라우팅은 VXLAN과 어떻게 다르고, 어떤 환경에 더 적합한가?
- `ip route`와 `bridge fdb`로 CNI가 설정한 라우팅 테이블을 어떻게 확인하는가?
- CNI 플러그인 장애 시 어떤 증상이 나타나고 어떻게 진단하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

파드 간 통신이 안 될 때 "CNI 문제인가, Network Policy 문제인가, 애플리케이션 문제인가"를 구분하지 못하면 진단에 수 시간이 걸린다. CNI 플러그인이 무엇을 설정하는지 알면 veth, 브리지, 라우팅 테이블, 터널 인터페이스 중 어디가 문제인지 빠르게 찾을 수 있다.

또한 Flannel과 Calico는 성능과 기능이 크게 다르다. Network Policy를 써야 한다면 Flannel 단독으로는 불가능하고, BGP 라우팅 환경이라면 Calico가 적합하다. 올바른 CNI를 선택하려면 내부 동작을 알아야 한다.

> 🔗 **network-deep-dive 연결:** VXLAN 캡슐화 원리와 BGP 라우팅 프로토콜은 network-deep-dive에서 다뤘다. 여기서는 그 개념이 쿠버네티스 CNI에서 어떻게 활용되는지 집중한다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: CNI 플러그인을 업그레이드 후 파드 간 통신 간헐 실패

원리를 모를 때의 판단:
  "CNI 재설치하면 되겠지" → kubectl delete daemonset flannel → 재설치
  → 재설치 중 기존 파드들도 통신 불가

실제 원인:
  VXLAN FDB(Forwarding Database) 테이블이 CNI 업그레이드 후 일부 노드에서
  stale 엔트리 유지 → 특정 노드로의 패킷이 잘못된 VTEP으로 전송

진단 방법:
  bridge fdb show dev flannel.1 | grep <대상 노드 IP>
  → 잘못된 MAC 주소 확인
  → 특정 노드의 CNI 파드 재시작으로 FDB 갱신
```

---

## ✨ 올바른 접근 (After — CNI 내부를 알고 난 진단)

```
CNI 문제 진단 체크리스트:

1. CNI 파드 상태
   kubectl get pods -n kube-system | grep -E "flannel|calico|cilium"
   → 모든 노드에 1개씩 Running?

2. 파드 IP 할당 여부
   kubectl get pod -o wide | grep <미배정 파드>
   → IP 없음 = CNI가 IP 할당 실패

3. CNI 설정 파일
   cat /etc/cni/net.d/*.conf
   → CNI 플러그인이 올바른 CIDR, 인터페이스 사용하는가?

4. 터널 인터페이스 (Flannel VXLAN)
   ip link show flannel.1
   ip -d link show flannel.1  ← VXLAN 설정 확인

5. FDB 테이블
   bridge fdb show dev flannel.1
   → 각 원격 VTEP의 MAC 주소 올바른가?

6. 노드 라우팅 테이블
   ip route show | grep 10.244
   → 원격 파드 CIDR 경로 있는가?
```

---

## 🔬 내부 동작 원리

### CNI 플러그인 호출 방식

CNI는 kubelet이 컨테이너 네트워크 설정을 외부 플러그인에 위임하는 표준 인터페이스다.

```
kubelet이 파드 생성 시 CNI 호출:

  1. /etc/cni/net.d/ 디렉토리에서 CNI 설정 파일 로드
     (알파벳 순서로 첫 번째 파일 사용)
     예: 10-flannel.conflist, 10-calico.conflist

  2. 설정 파일의 "type" 필드로 플러그인 바이너리 찾기
     /opt/cni/bin/flannel  또는  /opt/cni/bin/calico

  3. 플러그인 실행 (환경변수로 파라미터 전달):
     CNI_COMMAND=ADD         ← 네트워크 추가
     CNI_CONTAINERID=<id>    ← 컨테이너 ID
     CNI_NETNS=/proc/<pid>/ns/net  ← pause 컨테이너의 네트워크 namespace
     CNI_IFNAME=eth0         ← 컨테이너 내 인터페이스 이름

  4. 플러그인 작업:
     - 파드 IP 할당 (IPAM: IP Address Management)
     - veth pair 생성 (한쪽: 파드 namespace, 다른쪽: 호스트)
     - 라우팅 설정
     - 결과를 JSON으로 반환

  5. CNI_COMMAND=DEL (파드 삭제 시)
     - IP 반환, veth 삭제
```

### Flannel — VXLAN 오버레이 방식

Flannel은 가장 단순한 CNI로, VXLAN(Virtual Extensible LAN)으로 노드 간 파드 통신을 구현한다.

```
Flannel 설정 구조 (각 노드에 DaemonSet으로 실행):

  노드 1                          노드 2
  ┌───────────────────────────┐   ┌───────────────────────────┐
  │ 파드 CIDR: 10.244.1.0/24  │   │ 파드 CIDR: 10.244.2.0/24  │
  │                           │   │                           │
  │ ┌────────────┐            │   │ ┌────────────┐            │
  │ │ pod-a      │            │   │ │ pod-c      │            │
  │ │ 10.244.1.5 │            │   │ │ 10.244.2.7 │            │
  │ └──────┬─────┘            │   │ └──────┬─────┘            │
  │        │ veth             │   │        │ veth             │
  │   cni0 bridge             │   │   cni0 bridge             │
  │   (10.244.1.1)            │   │   (10.244.2.1)            │
  │        │                  │   │        │                  │
  │   flannel.1 (VTEP)        │   │   flannel.1 (VTEP)        │
  │   MAC: aa:bb:cc:dd:ee:01  │   │   MAC: aa:bb:cc:dd:ee:02  │
  └──────────┬────────────────┘   └──────────┬────────────────┘
             │ eth0 (192.168.1.10)           │ eth0 (192.168.1.11)
             └───────────── 물리 네트워크 ────┘
```

**VXLAN 캡슐화 동작**

```
파드 A (10.244.1.5) → 파드 C (10.244.2.7) 패킷:

  1. 파드 A에서 패킷 출발
     src: 10.244.1.5, dst: 10.244.2.7

  2. 노드1 라우팅 테이블:
     10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
     → flannel.1(VTEP)으로 전달

  3. flannel.1이 패킷 캡슐화 (VXLAN UDP 캡슐화):
     Outer Ethernet: src MAC(aa:bb:cc:dd:ee:01) → dst MAC(aa:bb:cc:dd:ee:02)
     Outer IP:       src 192.168.1.10 (노드1) → dst 192.168.1.11 (노드2)
     UDP:            8472 포트 (VXLAN 기본)
     VXLAN Header:   VNI(가상 네트워크 ID)
     Inner:          원래 패킷 (10.244.1.5 → 10.244.2.7)

  4. 물리 네트워크를 통해 노드2로 전송

  5. 노드2의 flannel.1이 패킷 언캡슐화
     → cni0 브리지로 전달 → 파드 C 수신

오버헤드:
  VXLAN 헤더: 50바이트 추가
  1500바이트 MTU 환경에서 실질 페이로드: 1450바이트
  MTU 설정 주의 필요
```

**Flannel이 MAC 주소를 아는 방법 (FDB)**

```
flannel DaemonSet이 각 노드의 파드 CIDR과 VTEP MAC을 수집:
  → etcd 또는 Kubernetes API에 저장
  → 각 노드의 Flannel이 이를 읽어 FDB 테이블 구성

bridge fdb show dev flannel.1:
  aa:bb:cc:dd:ee:02 dev flannel.1 dst 192.168.1.11 self permanent
  ↑                               ↑
  노드2 VTEP MAC                  노드2 물리 IP
```

### Calico — BGP 라우팅 방식

Calico는 BGP(Border Gateway Protocol)를 사용해 각 노드가 자신의 파드 CIDR을 다른 노드에게 광고한다. 이를 통해 VXLAN 캡슐화 없이 직접 라우팅한다.

```
Calico 설정 구조:

  노드 1 (AS 64512)               노드 2 (AS 64512)
  파드 CIDR: 10.244.1.0/24        파드 CIDR: 10.244.2.0/24
  ┌───────────────────────┐       ┌───────────────────────┐
  │ pod-a (10.244.1.5)    │       │ pod-c (10.244.2.7)    │
  │   eth0 ↕ veth         │       │   eth0 ↕ veth         │
  │                       │       │                       │
  │ BIRD BGP daemon       │◄─────►│ BIRD BGP daemon       │
  │ (Calico 내장)          │ BGP   │ (Calico 내장)          │
  │                       │peering│                       │
  │ 라우팅 테이블:           │       │ 라우팅 테이블:            │
  │ 10.244.2.0/24 via     │       │ 10.244.1.0/24 via     │
  │   192.168.1.11        │       │   192.168.1.10        │
  └───────────────────────┘       └───────────────────────┘

패킷 흐름 (VXLAN 없음):
  pod-a (10.244.1.5) → eth0 → veth
  → 노드1 라우팅: 10.244.2.0/24 via 192.168.1.11
  → 물리 네트워크 직접 전송 (캡슐화 없음)
  → 노드2: 10.244.2.0/24는 local → pod-c 수신
```

**Calico + Network Policy**

Calico는 Felix라는 에이전트가 iptables 또는 eBPF 규칙으로 Network Policy를 구현한다. Flannel 단독으로는 Network Policy를 지원하지 않아, Flannel + Calico(정책만) 또는 Cilium 같은 다른 플러그인이 필요하다.

---

## 💻 실전 실험

### 1. CNI 설정 파일 확인

```bash
docker exec -it kind-worker bash

# CNI 설정 파일
cat /etc/cni/net.d/10-kindnet.conflist
# (Kind는 kindnet이라는 자체 CNI 사용)

# CNI 플러그인 바이너리 목록
ls /opt/cni/bin/
# bridge  dhcp  flannel  host-local  ipvlan  loopback  macvlan  ptp  vlan
```

### 2. Flannel VXLAN 인터페이스 확인 (Flannel 클러스터)

```bash
# VXLAN 터널 인터페이스
ip -d link show flannel.1
# flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     link/ether aa:bb:cc:dd:ee:01 brd ff:ff:ff:ff:ff:ff
#     vxlan id 1 local 192.168.1.10 dev eth0 srcport 0 0 dstport 8472

# FDB (원격 VTEP 정보)
bridge fdb show dev flannel.1
# aa:bb:cc:dd:ee:02 dev flannel.1 dst 192.168.1.11 self permanent
# → 노드2의 VTEP MAC과 물리 IP
```

### 3. 파드 IP 할당 과정 관찰

```bash
# 파드 생성 전후 CNI 로그 확인
docker exec -it kind-control-plane bash
journalctl -u kubelet | grep -i cni | tail -20

# CNI가 할당한 IP 확인
kubectl get pod new-pod -o jsonpath='{.status.podIP}'
```

### 4. 노드 라우팅 테이블로 CNI 설정 확인

```bash
docker exec -it kind-worker bash

# 파드 CIDR 라우팅 확인
ip route show | grep -E "10\.244|cni0"
# 10.244.0.0/24 via 172.18.0.2 dev eth0   ← 다른 노드 파드 CIDR
# 10.244.1.0/24 dev cni0 proto kernel      ← 현재 노드 파드 CIDR (direct)
# 10.244.1.5 dev vethxxxx scope link       ← 개별 파드 경로

# ARP 테이블 (브리지가 파드 MAC 주소 학습)
ip neigh show dev cni0
# 10.244.1.5 dev cni0 lladdr xx:xx:xx:xx:xx:xx REACHABLE
```

### 5. 파드 간 통신 경로 패킷 캡처

```bash
# 노드에서 VXLAN 트래픽 캡처
docker exec -it kind-worker bash
tcpdump -i eth0 -n udp port 8472 -c 10

# 캡처하는 동안 다른 노드의 파드로 통신
kubectl exec pod-a -- ping 10.244.2.7 -c 3

# 출력: VXLAN 캡슐화된 패킷 (UDP/8472)
# 14:00:00.000000 IP 192.168.1.10.xxxxx > 192.168.1.11.8472: VXLAN(VNI=1)
```

---

## 📊 주요 CNI 플러그인 비교

| 항목 | Flannel | Calico | Cilium |
|-----|--------|--------|--------|
| 노드 간 통신 | VXLAN 오버레이 | BGP 직접 라우팅 | VXLAN 또는 eBPF |
| Network Policy | ❌ (단독) | ✅ (iptables/eBPF) | ✅ (eBPF) |
| 성능 | 중간 (캡슐화 오버헤드) | 높음 (직접 라우팅) | 높음 (eBPF 커널 레벨) |
| 설정 복잡도 | 낮음 | 중간 | 높음 |
| 클라우드 BGP 요구 | 없음 | AWS: VPC BGP, GCP: 가능 | 없음 |
| 적합 환경 | 소규모, 단순 | 중대규모, 정책 필요 | 대규모, 고성능, 보안 |

---

## ⚖️ 트레이드오프

**VXLAN vs BGP 라우팅**

VXLAN은 물리 네트워크 설정 변경 없이 동작하므로 설정이 단순하다. 단, 패킷 캡슐화로 50바이트 오버헤드가 발생하고 MTU 설정에 주의해야 한다. BGP 라우팅은 오버헤드가 없어 성능이 높지만, 클라우드 환경에서 VPC 라우팅 테이블이나 BGP peering 설정이 필요하다. AWS EKS에서 Calico BGP 모드를 쓰려면 VPC CNI 대신 직접 구성해야 한다.

**CNI 교체의 어려움**

클러스터 운영 중 CNI 플러그인을 교체하는 것은 매우 어렵다. 모든 노드의 기존 파드를 재생성해야 하고, 서비스 중단이 불가피하다. 초기 설계 시 Network Policy 필요성, 성능 요구사항, 클라우드 환경을 고려해 CNI를 선택해야 한다.

---

## 📌 핵심 정리

```
CNI 역할:
  파드 생성 시 kubelet이 CNI 바이너리 실행
  veth pair 생성, IP 할당, 라우팅 설정
  /etc/cni/net.d/ 설정 + /opt/cni/bin/ 바이너리

Flannel(VXLAN):
  각 노드가 VXLAN VTEP(flannel.1) 인터페이스 보유
  노드 간 패킷을 UDP/8472로 캡슐화해 전송
  FDB 테이블로 원격 VTEP MAC-IP 매핑 관리
  설정 단순, Network Policy 미지원

Calico(BGP):
  BIRD BGP 데몬으로 노드 간 파드 CIDR 경로 교환
  캡슐화 없이 직접 L3 라우팅 → 오버헤드 없음
  Felix 에이전트로 iptables/eBPF Network Policy 구현
  클라우드 BGP peering 설정 필요할 수 있음
```

---

## 🤔 생각해볼 문제

**Q1.** Flannel VXLAN 환경에서 파드 MTU를 1500으로 설정하면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

VXLAN 헤더가 50바이트를 추가하므로, 파드에서 1500바이트 패킷을 보내면 VXLAN 캡슐화 후 1550바이트가 된다. 물리 네트워크의 MTU가 1500이면 패킷이 너무 커서 단편화(fragmentation)가 발생하거나, DF(Don't Fragment) 비트가 설정된 경우 패킷이 드롭된다. 따라서 Flannel은 파드의 eth0 MTU를 1450(= 1500 - 50)으로 설정해야 한다. CNI 설정에 `mtu: 1450`이 없으면 대용량 패킷에서 간헐적인 통신 실패가 발생할 수 있다.

</details>

**Q2.** 클러스터에 새 노드를 추가했는데, 새 노드의 파드가 기존 노드의 파드와 통신이 안 된다. Flannel 환경에서 어떤 순서로 진단하는가?

<details>
<summary>해설 보기</summary>

(1) 새 노드에 Flannel DaemonSet Pod가 Running인지 확인. (2) 새 노드의 `flannel.1` 인터페이스가 생성됐는지 확인(`ip link show flannel.1`). (3) 기존 노드의 FDB 테이블에 새 노드의 VTEP 정보가 추가됐는지 확인(`bridge fdb show dev flannel.1`). (4) 새 노드의 라우팅 테이블에 기존 노드 파드 CIDR 경로가 있는지 확인(`ip route show | grep 10.244`). (5) UDP 8472 포트가 노드 간 방화벽에서 허용되는지 확인.

</details>

**Q3.** Cilium이 Flannel/Calico보다 고성능인 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

Cilium은 eBPF(extended Berkeley Packet Filter)를 사용해 패킷 처리를 커널의 XDP(eXpress Data Path) 또는 TC(Traffic Control) 레이어에서 수행한다. 기존 iptables는 패킷이 네트워크 스택을 전부 거쳐야 하는 반면, eBPF는 네트워크 스택 초기 단계에서 패킷을 처리한다. 이로 인해 컨텍스트 스위치가 줄고, iptables 규칙 수에 비례하는 성능 저하가 없다. 수만 개의 Service가 있는 대규모 클러스터에서 iptables 기반 kube-proxy는 성능 저하가 심하지만, Cilium의 eBPF 기반 구현은 Service 수와 무관하게 일정한 성능을 유지한다.

</details>

---

<div align="center">

**[⬅️ 이전: 파드 네트워킹 기초 — IP-per-Pod 모델](./01-pod-networking-basics.md)** | **[홈으로 🏠](../README.md)** | **[다음: Service와 kube-proxy — iptables 규칙 생성 ➡️](./03-service-kube-proxy.md)**

</div>
