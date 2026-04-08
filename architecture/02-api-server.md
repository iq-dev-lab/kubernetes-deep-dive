# API Server — 모든 요청의 진입점

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `kubectl apply`가 실행될 때 API Server 내부에서 어떤 파이프라인이 작동하는가?
- 인증(Authentication)과 인가(Authorization)는 어떻게 다르고, 각각 무엇을 검사하는가?
- Mutating Admission과 Validating Admission은 왜 순서가 중요한가?
- Watch API는 어떻게 컴포넌트들에게 이벤트를 실시간으로 전달하는가?
- `--v=8` 플래그로 kubectl의 실제 API 호출을 어떻게 추적하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

파드 생성이 거부됐을 때 이유를 모르는 경우가 많다. "권한이 없다"는 에러인지, Admission Controller가 거부한 것인지, 아니면 스펙 자체가 유효하지 않은 건지 구분이 안 된다. API Server의 요청 파이프라인을 알면 에러 메시지만 봐도 어느 단계에서 막혔는지 즉시 판단할 수 있다.

또한 Istio의 Sidecar 자동 주입, PodSecurityPolicy(또는 PSA), OPA Gatekeeper처럼 클러스터 동작을 커스터마이징하는 도구들이 모두 Admission Controller 메커니즘 위에서 동작한다. 내부를 모르면 이 도구들이 왜 특정 상황에서 예상과 다르게 동작하는지 설명할 수 없다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: kubectl apply 시 아래 에러 발생
  Error from server (Forbidden): error when creating "pod.yaml":
  pods "my-app" is forbidden: unable to validate against any security policy

원리를 모를 때의 대응:
  "권한 문제인가?" → ClusterRoleBinding 추가
  → 동일 에러 반복
  "RBAC 설정이 잘못됐나?" → 관리자에게 문의
  → 관리자도 RBAC은 정상이라고 함

실제 원인:
  RBAC(인가)은 통과했지만
  Admission Controller(PodSecurityAdmission)가 거부한 것
  에러 메시지의 "unable to validate against any security policy"가
  Validating Admission 단계임을 알았다면 즉시 PSA 정책을 확인했을 것
```

---

## ✨ 올바른 접근 (After — 파이프라인을 알고 난 진단)

```
API Server 요청 파이프라인:
  1. 인증(Authentication)   → 403: 인증 실패
  2. 인가(Authorization)    → 403 Forbidden: RBAC 거부
  3. Admission Controller   → 거부 메시지에 Webhook 이름 포함
  4. 스펙 유효성 검사       → 422 Unprocessable Entity

에러 유형으로 단계 특정:
  "Unauthorized (401)"          → 1단계: 인증 실패 → kubeconfig 확인
  "Forbidden (403)"             → 2단계: RBAC 미통과 → ClusterRoleBinding 확인
  "admission webhook ... denied" → 3단계: Webhook 거부 → Webhook 정책 확인
  "Invalid value"               → 4단계: 스펙 오류 → YAML 필드 확인
```

---

## 🔬 내부 동작 원리

### 요청 처리 파이프라인 전체 흐름

```
kubectl apply -f pod.yaml
      │
      ▼ HTTPS 요청
┌─────────────────────────────────────────┐
│             API Server                   │
│                                          │
│  1. 인증 (Authentication)                │
│     ├── X.509 클라이언트 인증서           │
│     ├── Bearer Token (ServiceAccount)    │
│     ├── OIDC (OpenID Connect)            │
│     └── Webhook 인증                     │
│                   │ 실패: 401            │
│                   ▼                      │
│  2. 인가 (Authorization)                 │
│     ├── RBAC: Role/ClusterRole 확인      │
│     ├── ABAC (거의 사용 안 함)           │
│     └── Webhook 인가                     │
│                   │ 실패: 403            │
│                   ▼                      │
│  3. Admission Controller                 │
│     ├── Mutating Webhook (수정)          │
│     │   예) Istio Sidecar 주입,          │
│     │       기본 리소스 한도 주입         │
│     ├── 오브젝트 스키마 유효성 검사       │
│     └── Validating Webhook (검증)        │
│         예) PodSecurity, OPA Gatekeeper  │
│                   │ 실패: 거부 메시지    │
│                   ▼                      │
│  4. etcd에 저장                          │
│     └── 성공: 201 Created               │
└─────────────────────────────────────────┘
```

### 인증(Authentication) 단계

API Server는 요청자가 누구인지 확인한다. "누구인가"를 판단하지, "무엇을 할 수 있는가"는 다음 단계의 역할이다.

**X.509 클라이언트 인증서**

`~/.kube/config`의 `client-certificate`와 `client-key`로 TLS 핸드셰이크를 수행한다. 인증서의 `CN(Common Name)` 필드가 사용자 이름이 되고, `O(Organization)` 필드가 그룹 이름이 된다.

```bash
# kubeconfig의 인증 정보 확인
kubectl config view --raw | grep client-certificate-data | \
  head -1 | awk '{print $2}' | base64 -d | openssl x509 -text -noout | grep -A2 "Subject:"
# 출력: Subject: O=system:masters, CN=kubernetes-admin
#       → 그룹: system:masters (cluster-admin 역할)
#       → 사용자: kubernetes-admin
```

**ServiceAccount Token**

파드 내에서 API Server에 접근할 때 사용한다. 파드의 `/var/run/secrets/kubernetes.io/serviceaccount/token`에 자동으로 마운트된다. 쿠버네티스 1.21부터는 만료 시간이 있는 Projected ServiceAccount Token을 사용한다.

```bash
# 파드 내에서 API Server 호출
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/default/pods \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### 인가(Authorization) 단계 — RBAC

인증이 완료되면 "이 사용자가 이 작업을 할 수 있는가"를 확인한다. 가장 널리 사용되는 방식은 RBAC(Role-Based Access Control)이다.

```
RBAC 구성 요소:
  Role          → 특정 네임스페이스에서 허용할 동작 정의
  ClusterRole   → 클러스터 전체 범위의 동작 정의
  RoleBinding   → 사용자/그룹/ServiceAccount에 Role 연결
  ClusterRoleBinding → 클러스터 전체 범위로 ClusterRole 연결

예시: default 네임스페이스의 파드를 읽을 수 있는 권한
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: default
    name: pod-reader
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

```bash
# 현재 사용자의 권한 확인
kubectl auth can-i create deployments
# yes

kubectl auth can-i create deployments --as=system:serviceaccount:default:my-sa
# no  ← 특정 ServiceAccount의 권한 테스트
```

### Admission Controller 단계

인가까지 통과한 요청에 대해 Admission Controller가 작동한다. 크게 두 종류로 나뉜다.

**Mutating Admission Webhook (수정)**

리소스 스펙을 변경할 수 있다. 요청된 오브젝트를 JSON Patch로 수정한 결과를 API Server에 반환한다. 대표적인 사용 사례:

- Istio: 파드 스펙에 `istio-proxy` 컨테이너 추가
- LimitRanger: Request/Limit 미설정 파드에 기본값 주입
- PodPreset: 공통 환경변수나 볼륨 자동 주입

**Validating Admission Webhook (검증)**

수정 없이 요청을 허용할지 거부할지만 결정한다. Mutating 이후에 실행된다. 순서가 중요한 이유는, Mutating이 스펙을 수정한 최종 상태를 Validating이 검사해야 하기 때문이다. 만약 순서가 반대라면 Validating이 통과시킨 스펙을 Mutating이 변경해 유효하지 않은 스펙이 저장될 수 있다.

```bash
# 현재 클러스터에 등록된 Webhook 확인
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# 예시 출력:
# NAME                                   WEBHOOKS   AGE
# istio-sidecar-injector                 1          10d
# cert-manager-webhook                   1          10d
```

### Watch 메커니즘의 구현

API Server는 etcd에서 변경 이벤트를 Watch하고, 이를 클라이언트에게 전달한다.

```
클라이언트(Scheduler 등)
  → GET /api/v1/pods?watch=true&resourceVersion=12345
  ← HTTP 200 (연결 유지, chunked transfer encoding)
  ← {"type":"ADDED","object":{...}}     ← 새 파드 생성 시
  ← {"type":"MODIFIED","object":{...}}  ← 파드 상태 변경 시
  ← {"type":"DELETED","object":{...}}   ← 파드 삭제 시
```

`resourceVersion`은 etcd의 수정 버전 번호다. 클라이언트가 재시작되어도 마지막으로 처리한 `resourceVersion`부터 이어받아 누락 없이 이벤트를 받을 수 있다.

**Watch Cache**

API Server는 etcd에서 Watch한 내용을 인메모리 캐시로 유지한다. 클라이언트의 `kubectl get pods` 같은 List 요청은 대부분 이 캐시에서 응답하므로 etcd 부하가 줄어든다.

---

## 💻 실전 실험

### 1. `--v=8`로 kubectl의 실제 HTTP 요청 추적

```bash
# -v=8: HTTP 헤더와 요청/응답 바디까지 출력
kubectl get pods -v=8 2>&1 | head -50

# 주요 출력:
# I0101 00:00:00] Config loaded from file: /root/.kube/config
# I0101 00:00:00] GET https://127.0.0.1:6443/api/v1/namespaces/default/pods
# I0101 00:00:00] Request Headers:
#     Authorization: Bearer eyJhbGci...  ← Bearer Token 인증
#     Accept: application/json
# I0101 00:00:00] Response Status: 200 OK

# apply의 실제 API 호출 추적
kubectl apply -f pod.yaml -v=8 2>&1 | grep -E "GET|POST|PUT|PATCH"
# GET /openapi/v2              ← 스펙 확인
# GET /api/v1/namespaces/...   ← 기존 리소스 조회 (존재 여부 확인)
# POST /api/v1/namespaces/.../pods ← 새 리소스 생성
# (이미 존재하면 PATCH로 수정)
```

### 2. API Server 감사 로그 확인

```bash
# API Server 요청 이벤트 스트림
kubectl get events --sort-by='.metadata.creationTimestamp' -A | head -20

# API Server 로그에서 요청 추적
kubectl logs -n kube-system kube-apiserver-kind-control-plane | grep "verb=\"create\"" | tail -10
```

### 3. RBAC 권한 실험

```bash
# 권한 없는 ServiceAccount 생성
kubectl create serviceaccount restricted-sa

# restricted-sa로 파드 목록 조회 시도
kubectl get pods --as=system:serviceaccount:default:restricted-sa
# Error from server (Forbidden): pods is forbidden

# Role과 RoleBinding 생성
kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding sa-pod-reader \
  --role=pod-reader \
  --serviceaccount=default:restricted-sa

# 권한 부여 후 재시도
kubectl get pods --as=system:serviceaccount:default:restricted-sa
# 성공
```

### 4. Admission Controller 동작 확인

```bash
# LimitRange로 기본 Resource Limit 설정
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - default:
      memory: 256Mi
      cpu: 200m
    defaultRequest:
      memory: 128Mi
      cpu: 100m
    type: Container
EOF

# Limit 미설정 파드 생성
kubectl run no-limit-pod --image=nginx

# Mutating Admission이 자동으로 기본값 주입했는지 확인
kubectl get pod no-limit-pod -o jsonpath='{.spec.containers[0].resources}'
# {"limits":{"cpu":"200m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}
# → LimitRange가 자동으로 주입한 것
```

---

## 📊 인증 방식별 비교

| 인증 방식 | 주요 사용 사례 | 토큰 만료 | 관리 복잡도 |
|---------|-------------|---------|------------|
| X.509 인증서 | 관리자 kubectl, 컴포넌트 간 통신 | 인증서 유효기간 | 인증서 갱신 필요 |
| ServiceAccount Token | 파드 내 API 접근 | 1시간 (기본) | 자동 갱신 |
| OIDC | SSO 연동 (Keycloak, Dex) | IdP 설정에 따름 | IdP 연동 필요 |
| Webhook 인증 | 외부 인증 시스템 연동 | 외부 시스템 의존 | 높음 |

---

## ⚖️ 트레이드오프

**Admission Webhook의 장점**
- 클러스터 동작을 코드 변경 없이 커스터마이징 가능
- Istio, Cert-Manager, OPA Gatekeeper 같은 강력한 도구의 기반

**Admission Webhook의 단점**
- Webhook 서버가 죽으면 `failurePolicy: Fail` 설정 시 파드 생성 자체가 불가능해짐
- Webhook 응답 지연이 모든 리소스 생성 지연으로 이어짐
- `failurePolicy: Ignore`로 설정하면 보안 정책이 우회될 수 있음

```bash
# Webhook 서버 장애 시 영향 확인
kubectl get mutatingwebhookconfigurations -o jsonpath='{.items[*].webhooks[*].failurePolicy}'
# Fail → Webhook 장애 시 리소스 생성 차단
# Ignore → Webhook 장애 시 정책 우회
```

---

## 📌 핵심 정리

```
API Server 요청 파이프라인 (순서가 중요):
  인증 → 인가(RBAC) → Mutating Admission → 스키마 검증 → Validating Admission → etcd 저장

에러로 단계 특정:
  401 Unauthorized     → 인증 실패
  403 Forbidden        → RBAC 거부
  admission webhook denied → Admission 거부
  422 Unprocessable    → 스펙 유효성 실패

Watch API:
  ?watch=true 파라미터로 HTTP 스트리밍 연결 유지
  resourceVersion으로 누락 없이 이벤트 이어받기
  API Server의 Watch Cache가 etcd 부하를 완충

Mutating → Validating 순서인 이유:
  수정된 최종 스펙을 검증해야 하기 때문
  순서가 반대면 검증을 통과한 스펙이 수정되어 무의미해짐
```

---

## 🤔 생각해볼 문제

**Q1.** `kubectl apply`와 `kubectl create`의 차이는 API Server에서 어떻게 드러나는가?

<details>
<summary>해설 보기</summary>

`kubectl create`는 리소스가 이미 존재하면 실패한다. 내부적으로 POST 요청만 보낸다. `kubectl apply`는 리소스가 없으면 POST(생성), 이미 있으면 PATCH(수정)를 보낸다. `--v=8` 플래그로 확인하면 apply는 먼저 GET으로 존재 여부를 확인하고, 그 결과에 따라 POST 또는 PATCH를 선택한다.

</details>

**Q2.** 클러스터에 Mutating Webhook이 10개 등록되어 있다. 파드 생성 시 어떤 순서로 호출되는가?

<details>
<summary>해설 보기</summary>

Mutating Webhook은 병렬로 호출되는 것이 기본이 아니다. `reinvocationPolicy` 설정에 따라 다르다. 기본적으로 각 Webhook은 순서대로 호출되며, 한 Webhook의 결과(수정된 스펙)가 다음 Webhook의 입력이 된다. `reinvocationPolicy: IfNeeded`로 설정하면 다른 Webhook이 스펙을 수정한 경우 해당 Webhook이 다시 호출된다. 이 때문에 Webhook 간 의존 관계에 주의해야 한다.

</details>

**Q3.** ServiceAccount Token이 자동으로 파드에 마운트되는데, 이를 비활성화하려면 어떻게 하는가? 왜 비활성화가 필요할 수 있는가?

<details>
<summary>해설 보기</summary>

파드 스펙에 `automountServiceAccountToken: false`를 설정하거나, ServiceAccount 오브젝트에 같은 설정을 추가하면 된다. 파드 내 프로세스가 API Server에 접근할 필요가 없는 경우, 토큰이 마운트되면 취약점 발생 시 공격자가 API Server에 접근할 수 있는 경로가 된다. 최소 권한 원칙에 따라 API 접근이 불필요한 파드에는 비활성화하는 것이 권장된다.

</details>

---

> ⬅️ 이전: [01. 클러스터 아키텍처 개요](./01-cluster-architecture-overview.md)  
> ➡️ 다음: [03. etcd와 Raft 합의 알고리즘](./03-etcd-raft.md)
