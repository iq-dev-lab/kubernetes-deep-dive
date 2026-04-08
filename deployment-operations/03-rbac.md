# RBAC — 최소 권한 원칙 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Role, ClusterRole, RoleBinding, ClusterRoleBinding의 차이와 조합 방법은?
- ServiceAccount가 파드에 마운트된 토큰으로 API Server에 인증하는 원리는?
- `kubectl auth can-i`로 어떤 권한을 어떻게 확인하는가?
- 최소 권한 원칙 구현 시 가장 자주 빠지는 함정은 무엇인가?
- Operator나 Custom Controller가 필요한 RBAC 권한을 어떻게 설계하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"파드 안에서 kubectl을 쓸 수 없다"거나 "API Server 접근이 거부된다"는 문제는 대부분 RBAC 설정 오류다. 반대로 보안을 소홀히 하면 파드 하나가 탈취됐을 때 클러스터 전체가 위험에 노출된다. 최소 권한 원칙을 지키면 공격 범위를 해당 파드의 권한으로 제한할 수 있다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: 개발 중 RBAC 오류가 반복 발생
      → cluster-admin ClusterRoleBinding을 파드에 부여

원리를 모를 때의 판단:
  "권한 문제니까 모든 권한 주면 되겠지"
  → serviceAccount에 cluster-admin 부여

실제 위험:
  해당 파드가 탈취되면 공격자가 cluster-admin 권한 획득
  → 모든 네임스페이스의 시크릿 열람
  → 다른 파드 삭제/생성
  → 악성 파드 배포로 노드 전체 장악 가능

올바른 접근:
  필요한 권한만 정확히 파악 후 Role 생성
  kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>
  → 필요한 것만 허용 (최소 권한 원칙)
```

---

## ✨ 올바른 접근 (After — 필요한 권한 분석 후 최소 부여)

```
RBAC 설계 순서:
  1. 파드가 어떤 API Server 리소스에 접근하는가?
     예: pods 조회, configmaps 읽기

  2. 어느 네임스페이스에서만 접근하는가?
     같은 ns만: Role + RoleBinding
     전체 클러스터: ClusterRole + ClusterRoleBinding

  3. 어떤 verb(동작)이 필요한가?
     get, list, watch (읽기)
     create, update, patch, delete (쓰기)

  4. Role 생성 후 ServiceAccount 생성 + RoleBinding

  5. 검증:
     kubectl auth can-i get pods \
       --as=system:serviceaccount:<ns>:<sa>
```

---

## 🔬 내부 동작 원리

### RBAC 계층 구조

```
Role (네임스페이스 범위)
  → "default 네임스페이스에서 pods를 get/list 가능"
  
ClusterRole (클러스터 전체 범위)
  → "모든 네임스페이스에서 nodes를 get 가능"
  → 또는 비네임스페이스 리소스: nodes, PV, StorageClass

RoleBinding (네임스페이스 범위)
  → Role 또는 ClusterRole을 특정 Subject에 바인딩
  → "이 네임스페이스에서만"

ClusterRoleBinding
  → ClusterRole을 전체 클러스터에서 Subject에 바인딩

조합:
  Role + RoleBinding: 한 ns의 리소스, 한 ns에서만
  ClusterRole + RoleBinding: 여러 ns의 공통 Role을 각 ns에 바인딩
    (ClusterRole을 "Role 템플릿"처럼 재사용)
  ClusterRole + ClusterRoleBinding: 전체 클러스터 권한
```

```
Subject (바인딩 대상):
  User:           외부 사용자 (kubeconfig의 user)
  Group:          사용자 그룹
  ServiceAccount: 파드가 API Server에 인증할 때 사용
```

### ServiceAccount 토큰 마운트 원리

```
파드에 ServiceAccount 지정:
  spec.serviceAccountName: my-sa  (미지정 시 default SA 사용)

kubelet이 파드 생성 시:
  TokenRequest API로 SA 토큰 요청 (만료 시간 있는 JWT)
  /var/run/secrets/kubernetes.io/serviceaccount/ 에 마운트:
    token      ← JWT Bearer Token
    ca.crt     ← API Server CA 인증서
    namespace  ← 현재 네임스페이스

파드 내부에서 API Server 접근:
  curl -H "Authorization: Bearer $(cat /var/run/secrets/...token)" \
       --cacert /var/run/secrets/.../ca.crt \
       https://kubernetes.default.svc/api/v1/pods

  API Server:
    1. 토큰 검증 (JWT 서명, 만료 시간)
    2. ServiceAccount 특정
    3. RBAC 권한 확인 (해당 SA에 binding된 Role)
    4. 허용/거부
```

### 비네임스페이스 리소스

```
네임스페이스 리소스 (Role로 제어):
  pods, services, endpoints, configmaps, secrets
  deployments, statefulsets, daemonsets (apps 그룹)
  ingresses (networking 그룹)

비네임스페이스 리소스 (ClusterRole만 가능):
  nodes, persistentvolumes, storageclasses
  clusterroles, clusterrolebindings
  namespaces

특수 리소스 (서브리소스):
  pods/log      → kubectl logs 권한
  pods/exec     → kubectl exec 권한
  pods/portforward → kubectl port-forward 권한
  deployments/scale → HPA가 파드 수 변경 권한
```

---

## 💻 실전 실험

### 1. RBAC 객체 생성 실습

```bash
# 1. ServiceAccount 생성
kubectl create serviceaccount pod-reader -n default

# 2. Role 생성 (default ns에서 pods 조회만)
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]    # kubectl logs 허용
  verbs: ["get"]
EOF

# 3. RoleBinding
kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=default:pod-reader \
  -n default
```

### 2. 권한 확인

```bash
# SA가 어떤 권한을 갖는지 확인
kubectl auth can-i get pods \
  --as=system:serviceaccount:default:pod-reader
# yes

kubectl auth can-i delete pods \
  --as=system:serviceaccount:default:pod-reader
# no

kubectl auth can-i get pods -n kube-system \
  --as=system:serviceaccount:default:pod-reader
# no  ← default ns만 허용했으므로

# 현재 사용자의 모든 권한 목록
kubectl auth can-i --list
```

### 3. 파드에서 API Server 접근 테스트

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: api-test
spec:
  serviceAccountName: pod-reader
  containers:
  - name: test
    image: curlimages/curl
    command: ['sh', '-c', 'sleep 3600']
EOF

# 파드 내부에서 API Server 접근
kubectl exec api-test -- sh -c '
  TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  curl -s --cacert $CACERT \
    -H "Authorization: Bearer $TOKEN" \
    https://kubernetes.default.svc/api/v1/namespaces/default/pods \
    | jq .items[].metadata.name
'
# "api-test"  ← 파드 목록 조회 성공

# 삭제 시도 (권한 없음)
kubectl exec api-test -- sh -c '
  TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  curl -s --cacert $CACERT \
    -H "Authorization: Bearer $TOKEN" \
    -X DELETE \
    https://kubernetes.default.svc/api/v1/namespaces/default/pods/api-test
'
# {"kind":"Status","code":403,...}  ← 권한 없음
```

### 4. Operator용 RBAC 설계

```yaml
# Operator가 자신의 CR과 연관 리소스를 관리하는 RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-operator
rules:
# CRD 리소스 관리
- apiGroups: ["mycompany.io"]
  resources: ["mydatabases", "mydatabases/status", "mydatabases/finalizers"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# StatefulSet, Service 생성 (DB 클러스터 구성)
- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# 이벤트 기록
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
```

---

## 📊 RBAC 조합 패턴 비교

| Role 유형 | Binding 유형 | 적용 범위 | 사용 케이스 |
|---------|-----------|---------|-----------|
| Role | RoleBinding | 단일 네임스페이스 | 일반 앱 서비스 |
| ClusterRole | RoleBinding | 특정 네임스페이스 (Role 재사용) | 멀티 ns 공통 역할 |
| ClusterRole | ClusterRoleBinding | 전체 클러스터 | Operator, Node Agent, 클러스터 도구 |

---

## ⚖️ 트레이드오프

**최소 권한 vs 개발 속도**

개발 초기에 최소 권한을 정확히 파악하려면 시간이 필요하다. 필요한 권한을 하나씩 추가하며 오류를 해결하는 방식보다 처음에 더 많은 분석이 필요하다. 그러나 `audit` 로그를 활용하면 실제 API 호출을 기반으로 필요한 권한 목록을 자동으로 파악할 수 있다. 프로덕션 전환 전 `kubectl auth can-i --list`로 부여된 권한을 검토하는 것이 필수다.

**default ServiceAccount 사용의 위험**

`serviceAccountName`을 지정하지 않으면 `default` SA가 사용된다. `default` SA에 ClusterRoleBinding이 붙어있는 경우, 모든 파드가 해당 권한을 갖게 된다. 모든 Deployment에 전용 ServiceAccount를 생성하고, `automountServiceAccountToken: false`로 API 접근이 불필요한 파드의 토큰 마운트를 비활성화하는 것이 권장된다.

---

## 📌 핵심 정리

```
4개 오브젝트:
  Role:               네임스페이스 범위 권한 정의
  ClusterRole:        클러스터 범위 권한 정의 (비네임스페이스 리소스 포함)
  RoleBinding:        Role/ClusterRole을 SA/User에 네임스페이스 범위 바인딩
  ClusterRoleBinding: ClusterRole을 SA/User에 전체 범위 바인딩

ServiceAccount 인증:
  /var/run/secrets/...token (JWT) → API Server Bearer Token
  API Server → RBAC 검사 → 허용/거부

검증:
  kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa>
  kubectl auth can-i --list → 현재 사용자 전체 권한

최소 권한 원칙:
  필요한 verb만 (get/list/watch vs create/delete)
  필요한 네임스페이스만
  apiGroup 정확히 지정 (빈 문자열 = core API)
```

---

## 🤔 생각해볼 문제

**Q1.** ClusterRole을 RoleBinding으로 바인딩하면 어떤 효과가 있는가?

<details>
<summary>해설 보기</summary>

RoleBinding은 바인딩이 적용되는 네임스페이스 범위로 권한을 제한한다. ClusterRole에 "모든 네임스페이스의 pods 조회" 권한이 있어도, RoleBinding으로 바인딩하면 "이 네임스페이스의 pods 조회"로 범위가 제한된다. 이를 통해 ClusterRole을 "권한 템플릿"처럼 재사용하면서도, 각 네임스페이스에 RoleBinding으로 적용해 범위를 제한할 수 있다. 멀티테넌트 환경에서 유용한 패턴이다.

</details>

**Q2.** `kubectl exec`, `kubectl logs`, `kubectl port-forward`를 사용자에게 허용하려면 어떤 RBAC를 설정해야 하는가?

<details>
<summary>해설 보기</summary>

각각 서브리소스 권한이 필요하다. `kubectl exec`는 `pods/exec` 서브리소스에 `create` verb, `kubectl logs`는 `pods/log` 서브리소스에 `get` verb, `kubectl port-forward`는 `pods/portforward` 서브리소스에 `create` verb가 필요하다. RBAC rules 예시: `- apiGroups: [""], resources: ["pods/exec", "pods/log", "pods/portforward"], verbs: ["create", "get"]`. 이 권한이 없으면 kubectl 명령이 권한 거부 오류를 반환한다.

</details>

**Q3.** HPA가 Deployment의 파드 수를 변경하는 데 필요한 RBAC 권한은 무엇인가?

<details>
<summary>해설 보기</summary>

HPA Controller는 `deployments/scale` 서브리소스에 `get`, `update` 권한이 필요하다. 기본적으로 쿠버네티스 시스템 컴포넌트는 `system:controller:horizontal-pod-autoscaler` ClusterRole을 통해 이 권한을 이미 가지고 있다. 별도로 설정할 필요는 없다. 단, 커스텀 메트릭을 사용하는 HPA의 경우 Prometheus Adapter의 ServiceAccount에 `custom.metrics.k8s.io` API 서버 접근 권한이 필요하다.

</details>

---

<div align="center">

**[⬅️ 이전: 무중단 배포 — Probe와 Hook의 삼각편대](./02-zero-downtime-deploy.md)** | **[홈으로 🏠](../README.md)** | **[다음: ConfigMap과 Secret — 설정 분리의 원리 ➡️](./04-configmap-secret.md)**

</div>
