# 클러스터 오토스케일러 — 노드 추가와 스케일다운

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 클러스터 오토스케일러는 어떤 조건에서 노드를 추가하고, 어떤 조건에서 제거하는가?
- Pending 파드를 감지해 노드를 추가하기까지 내부적으로 어떤 과정이 일어나는가?
- 스케일다운 시 파드를 다른 노드로 이동하는 메커니즘은 무엇인가?
- PodDisruptionBudget이 클러스터 오토스케일러에서 어떻게 최소 가용성을 보장하는가?
- 스케일다운이 일어나지 않을 때 어떤 파드가 원인인지 어떻게 찾는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

클라우드에서 쿠버네티스를 운영할 때 노드를 24/7 고정으로 유지하면 트래픽이 적은 시간에 비용이 낭비된다. 클러스터 오토스케일러는 수요에 맞게 노드를 자동으로 추가·제거한다. 단, "왜 노드가 줄어들지 않는가"는 실무에서 매우 흔한 문제다. 특정 파드가 노드를 "붙잡고" 있어 스케일다운을 막는 경우를 진단하려면 내부 동작을 알아야 한다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 트래픽이 줄었는데 노드가 줄어들지 않음
      새벽 2시 노드 40개 → 아침 9시에도 40개 유지
      불필요한 클라우드 비용 발생

원리를 모를 때의 판단:
  "CA 설정이 잘못됐나?" → CA 재배포
  → 동일 증상

실제 원인:
  kubectl get pods -A -o wide | grep <스케일다운 대상 노드>
  → worker-15 노드에만 배치된 특정 파드 발견
     kube-system의 DaemonSet은 당연히 있고...
     monitoring/grafana-0 : StatefulSet 파드 (PVC 있음)

  StatefulSet 파드 + PVC → 다른 노드로 이동 불가
  → CA가 해당 노드를 스케일다운 대상에서 제외

해결:
  StatefulSet 파드를 다른 노드로 수동 이동
  또는 해당 노드를 스케일다운 제외 대상에서 허용
```

---

## ✨ 올바른 접근 (After — CA 동작을 이해한 운영)

```
CA 스케일다운 불가 원인 진단:
  kubectl describe node <worker-15> | grep -i "scale-down"
  → "scale-down-disabled" 어노테이션?
  
  kubectl get pods -A --field-selector spec.nodeName=worker-15
  → 이동 불가 파드 목록 확인:
    - kube-system DaemonSet: 정상 (CA가 무시)
    - local-storage PVC 사용 파드: 이동 불가
    - PodDisruptionBudget 위반 파드: 이동 불가
    - static pod: 이동 불가

스케일다운 블로킹 파드 제거:
  StatefulSet 파드 → 다른 노드에 여유공간 확보 후 자동 이동 허용
  로컬 PV 파드 → 로컬 PV를 사용하는 한 노드 고정 (CA 한계)
```

---

## 🔬 내부 동작 원리

### 스케일아웃 — 노드 추가 흐름

```
Pending 파드 발생
  → Scheduler: Insufficient CPU/Memory → FailedScheduling 이벤트
      │
      ▼
CA가 Pending 파드 감지 (10초 주기)
  파드 FailedScheduling 이벤트 확인
  원인이 "리소스 부족"인 경우만 처리
  (Taint 불일치, Affinity 불일치는 노드 추가로 해결 안 됨)

      │
      ▼
노드 그룹 시뮬레이션
  현재 클러스터에 새 노드가 추가됐다고 가정
  각 노드 그룹(ASG/MIG)의 노드 타입으로 시뮬레이션 Scheduler 실행
  → 어느 노드 그룹에서 노드를 추가하면 Pending 파드가 배치 가능한가?

      │
      ▼
클라우드 API 호출
  AWS: AutoScalingGroup.SetDesiredCapacity() 증가
  GCP: InstanceGroupManager.Resize() 증가
  
  약 1~3분 후 새 노드 클러스터 조인

      │
      ▼
Scheduler가 새 노드에 Pending 파드 배치
```

```
CA 스케일아웃 소요 시간:
  Pending 감지:         0~10초
  노드 프로비저닝:       1~3분 (클라우드 VM 생성)
  노드 쿠버네티스 조인:  30초~2분
  파드 스케줄링:         수 초
  합계:                 약 2~5분
  
  → HPA로 파드 수가 먼저 늘어나고 CA가 노드를 추가하는 시간 동안
    새 파드들이 Pending 상태 유지
```

### 스케일다운 — 노드 제거 흐름

```
CA 스케일다운 조건 (기본값):
  노드 자원 사용률 < 50% (CPU+메모리 Request 합계 기준)
  10분 이상 지속

      │
      ▼
스케일다운 후보 노드 선정
  자원 사용률이 낮은 노드
  해당 노드의 파드를 다른 노드에 수용 가능한가? (시뮬레이션)

      │ 파드 이동 가능성 검사 (각 파드 하나씩):
      ▼
이동 불가 파드 확인:
  1. kube-system 파드 (일부)
     → DaemonSet: 모든 노드에 있으므로 문제 없음
     → Static Pod: 이동 불가 → 해당 노드 스케일다운 불가
  
  2. 로컬 스토리지 사용 파드
     → emptyDir, hostPath → 이동 시 데이터 소실 → 차단
     → 로컬 PV → 이동 불가 → 차단
  
  3. PodDisruptionBudget 위반
     → 파드를 Evict하면 PDB minAvailable 위반 → 차단
  
  4. strict node affinity
     → 이 파드는 이 노드에만 배치 가능 → 차단
  
  5. "cluster-autoscaler.kubernetes.io/safe-to-evict: false" 어노테이션
     → 명시적 이동 거부 → 차단

      │ 모든 파드 이동 가능한 경우
      ▼
파드 Drain (kubectl drain과 동일)
  각 파드에 Eviction 요청 (SIGTERM → graceful shutdown)
  PDB 위반 없는 범위에서 순서대로 이동
  새 노드에 재스케줄링

      │
      ▼
노드 삭제
  클라우드 API 호출: 인스턴스 종료
  Node 오브젝트 API Server에서 삭제
```

### PodDisruptionBudget — 스케일다운 중 최소 가용성 보장

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2        # 최소 2개 항상 실행
  # 또는
  # maxUnavailable: 1    # 최대 1개만 동시에 종료 허용
  selector:
    matchLabels:
      app: my-api
```

```
PDB가 없는 경우:
  CA 스케일다운 시 파드 3개 모두 한꺼번에 이동 가능
  → 이동 중 순간적으로 파드 0개 → 서비스 중단

PDB minAvailable: 2 적용:
  파드 3개 중 최소 2개 항상 유지
  → 한 번에 1개만 Evict 가능
  → 1개 새 노드에서 Running 확인 후 다음 Evict
  → 순차적 이동으로 서비스 중단 없음
  
  단, CA 스케일다운 속도가 느려짐 (각 파드 이동 후 대기)
```

### CA 동작 관찰 명령어

```bash
# CA 상태 확인
kubectl get configmap -n kube-system cluster-autoscaler-status -o yaml

# CA 로그 (스케일링 결정 이유)
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep -E "scale-up|scale-down|Pending" | tail -20

# 스케일다운 불가 노드 진단
kubectl describe node <node-name> | grep -i "safe-to-evict\|scale-down"

# 스케일다운 차단 파드 찾기
kubectl get pods -A --field-selector spec.nodeName=<node-name> \
  -o custom-columns=\
NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
ANNOTATIONS:.metadata.annotations
```

---

## 💻 실전 실험

### 1. CA 설치 (EKS 환경 예시)

```bash
# EKS IAM OIDC 설정 후
helm repo add autoscaler https://kubernetes.github.io/autoscaler

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=<cluster-name> \
  --set awsRegion=<region> \
  --set rbac.serviceAccount.annotations."eks.amazonaws.com/role-arn"=<role-arn>

# CA 파드 확인
kubectl get pods -n kube-system | grep cluster-autoscaler
```

### 2. 스케일아웃 트리거

```bash
# 현재 노드가 수용할 수 없는 대규모 Deployment
kubectl create deployment scale-test --image=nginx --replicas=20
kubectl set resources deployment scale-test --requests=cpu=500m,memory=512Mi

# Pending 파드 확인
kubectl get pods | grep Pending

# CA 로그 확인
kubectl logs -n kube-system deployment/cluster-autoscaler -f | grep scale-up
```

### 3. PDB 설정 후 스케일다운 관찰

```bash
# PDB 생성
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: scale-test-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: scale-test
EOF

# 스케일다운 시도 (Deployment replicas 줄이기)
kubectl scale deployment scale-test --replicas=3

# PDB 준수하며 파드 이동 관찰
kubectl get pods -w
kubectl get nodes -w
```

### 4. 스케일다운 방지 어노테이션

```yaml
# 특정 파드를 스케일다운에서 보호
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
```

```bash
# 노드를 CA 스케일다운에서 영구 제외
kubectl annotate node <node-name> \
  cluster-autoscaler.kubernetes.io/scale-down-disabled="true"
```

### 5. CA 상태 대시보드 확인

```bash
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml | \
  grep -A50 "status:"
# Cluster-autoscaler status:
#   NodeGroups:
#     Name: eks-nodegroup-xxx
#     Health: Healthy
#     ScaleUp: NoActivity
#     ScaleDown: NoCandidates
```

---

## 📊 스케일다운 차단 원인 비교

| 차단 원인 | 해결 방법 | 비고 |
|---------|---------|-----|
| 로컬 PV 파드 | 원격 스토리지로 전환 | 로컬 PV는 이동 불가 |
| PDB 위반 | 다른 노드 여유 확보 또는 PDB 조정 | 의도적 보호 |
| safe-to-evict: false | 어노테이션 제거 | 수동 설정 |
| Static Pod | 해당 없음 (control-plane) | 정상 동작 |
| emptyDir 파드 | 기본 설정은 이동 가능 | 설정에 따라 다름 |
| Strict NodeAffinity | Affinity 완화 또는 다른 노드 레이블 추가 | 배치 제약 |

---

## ⚖️ 트레이드오프

**스케일다운 지연 vs 비용 절감**

스케일다운 임계값(기본 50%)과 안정화 시간(기본 10분)을 낮추면 더 적극적으로 비용을 줄일 수 있지만, 트래픽 급증 시 노드 확보에 2~5분이 걸려 파드가 오래 Pending 상태가 된다. 스케일다운을 더 보수적으로(30%, 20분 안정화) 설정하면 비용 절감은 줄어들지만 안정성이 높아진다. 서비스 특성에 맞게 조정해야 한다.

**스팟 인스턴스와 CA**

비용 절감을 위해 AWS Spot 인스턴스나 GCP Preemptible VM을 사용할 때, 인스턴스가 갑자기 회수될 수 있다. CA는 이를 노드 삭제로 처리하고 다른 노드에 파드를 재스케줄링한다. PDB를 설정해 항상 최소 가용 파드를 유지하도록 하고, 중요 파드는 On-demand 인스턴스의 노드 그룹에만 배치하는 것이 권장된다.

---

## 📌 핵심 정리

```
CA 스케일아웃:
  Pending 파드(리소스 부족) 감지 → 노드 그룹 시뮬레이션
  → 클라우드 API 호출 → VM 생성 → 클러스터 조인 (총 2~5분)
  HPA와 함께 쓰면: HPA → 파드 수 증가 → Pending → CA → 노드 추가

CA 스케일다운:
  자원 사용률 < 50%, 10분 지속 → 후보 노드 선정
  파드 이동 가능성 검사 → 이동 가능한 경우 Drain
  PDB 준수하며 순차적 이동 → 노드 삭제

스케일다운 차단 원인:
  로컬 PV, safe-to-evict: false, PDB 위반, Static Pod
  kubectl describe node | grep scale-down 으로 진단

PDB 필수:
  스케일다운 중 최소 가용 파드 수 보장
  minAvailable: N 또는 maxUnavailable: M 설정
```

---

## 🤔 생각해볼 문제

**Q1.** HPA로 파드가 10개로 늘어난 후, 트래픽이 줄어 HPA가 파드를 5개로 줄였다. 그런데 CA는 여전히 노드를 줄이지 않는다. 어떤 상황인가?

<details>
<summary>해설 보기</summary>

CA의 스케일다운은 노드 자원 사용률이 10분 이상 낮아야 시작된다. HPA가 파드를 줄였다고 해도 CA가 즉시 반응하지 않는다. 또한 5개 파드가 여러 노드에 분산되어 있어, 어느 한 노드도 "파드를 다른 노드로 모두 이동시킬 수 있는" 상태가 아닐 수 있다. CA는 파드를 강제로 특정 노드에 몰아넣는 것이 아니라, 빈 노드(또는 이동 가능한 파드만 있는 노드)를 제거한다. 노드 자원 사용률이 임계값 아래로 떨어지고 10분이 지나면 스케일다운이 시작된다.

</details>

**Q2.** 클러스터에 여러 노드 그룹(m5.xlarge, c5.2xlarge)이 있을 때, CA는 어떤 노드 그룹의 노드를 추가하는가?

<details>
<summary>해설 보기</summary>

CA는 Expander 전략에 따라 노드 그룹을 선택한다. 기본 전략은 `random`이지만, 일반적으로 권장되는 것은 `least-waste`(파드 배치 후 낭비되는 자원이 가장 적은 노드 그룹 선택) 또는 `priority`(우선순위 기반)이다. `--expander=least-waste` 플래그로 설정한다. 예를 들어 Pending 파드가 CPU 2, 메모리 4GB를 필요로 한다면, m5.xlarge(4 CPU, 16GB) vs c5.2xlarge(8 CPU, 16GB) 중 낭비가 적은 m5.xlarge가 선택된다.

</details>

**Q3.** `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` 어노테이션이 달린 파드가 노드에 있다. 이 노드를 수동으로 drain하려 하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`kubectl drain` 명령도 기본적으로 이 어노테이션을 존중하고 해당 파드를 Evict하지 않아 drain이 완료되지 않는다. 강제로 drain하려면 `--disable-eviction` 또는 `--force` 플래그를 사용해야 한다. 이는 CA뿐만 아니라 수동 노드 유지보수 시에도 동일하게 적용된다. 따라서 `safe-to-evict: false`는 신중하게 사용해야 하며, 반드시 필요한 파드에만 적용해야 한다. 일반적으로 Prometheus 같은 모니터링 핵심 컴포넌트나 로컬 상태를 가진 파드에 적용된다.

</details>

---

> ⬅️ 이전: [04. VPA — Request/Limit 자동 조정](./04-vpa.md)  
> ➡️ 다음 챕터: [Ch6-01. Deployment 전략 — RollingUpdate와 Recreate](../deployment-operations/01-deployment-strategies.md)
