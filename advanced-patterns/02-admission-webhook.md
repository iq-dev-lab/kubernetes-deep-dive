# Admission Webhook — 리소스 생성/수정 인터셉트

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Mutating Webhook과 Validating Webhook은 API Server 파이프라인 어느 단계에서 호출되는가?
- Istio가 Envoy Sidecar를 자동 주입하는 내부 메커니즘은?
- Webhook의 TLS 인증서와 `caBundle` 설정이 왜 중요한가?
- Webhook 서버가 응답하지 않을 때 어떻게 되는가?
- `failurePolicy: Fail` vs `failurePolicy: Ignore`의 트레이드오프는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"kubectl apply를 했는데 갑자기 파드가 거부된다"거나 "배포할 때마다 컨테이너가 하나 더 생긴다"는 현상의 원인이 Admission Webhook이다. OPA/Gatekeeper로 보안 정책을 강제하거나, Istio가 Sidecar를 자동 주입하는 모든 기능이 Webhook 위에서 동작한다. Webhook 장애 진단은 Webhook을 이해해야만 가능하다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: kubectl apply -f deployment.yaml이 갑자기 실패
      "admission webhook denied the request"

원리를 모를 때의 판단:
  "YAML이 잘못됐나?" → 여러 번 확인 → 정상
  "클러스터 문제인가?" → 재시작 시도
  → 원인 모름

실제 원인:
  Validating Webhook이 특정 정책 위반으로 거부
  예: OPA Gatekeeper가 "모든 Deployment에 resource request 필수" 정책 적용
  → requests.cpu, requests.memory가 없는 Deployment 거부

진단 방법:
  kubectl apply -f deployment.yaml 오류 메시지 자세히 읽기
  → "denied by <webhook-name>: <이유>" 확인
  kubectl get validatingwebhookconfigurations
  kubectl describe validatingwebhookconfiguration <name>
  → rules 필드에서 어떤 리소스/verb에 적용되는지 확인
```

---

## ✨ 올바른 접근 (After — Webhook 파이프라인을 이해한 운영)

```
API Server 요청 처리 파이프라인:
  kubectl apply
    │
    ▼
  1. 인증 (Authentication)
    │
    ▼
  2. 인가 (Authorization / RBAC)
    │
    ▼
  3. Mutating Admission Webhooks (순서: webhook 목록 순)
     → 리소스 스펙 수정 (Sidecar 주입, default 값 추가)
    │
    ▼
  4. Object Schema Validation
    │
    ▼
  5. Validating Admission Webhooks (병렬 호출)
     → 정책 위반 시 거부 가능
    │
    ▼
  6. etcd 저장

Mutating이 먼저, Validating이 나중:
  이유: Mutating이 값을 추가한 후에 Validating이 완성된 스펙을 검증해야 함
  예: Mutating이 default memory request 추가 → Validating이 "request 있는가?" 검사
```

---

## 🔬 내부 동작 원리

### Webhook 등록 방법

```yaml
# Mutating Webhook 설정
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: istio-sidecar-injector
webhooks:
- name: sidecar-injector.istio.io
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
    operations: ["CREATE"]   # Pod 생성 시에만 호출
  namespaceSelector:
    matchLabels:
      istio-injection: enabled  # 이 레이블의 namespace에만 적용
  clientConfig:
    service:
      name: istiod
      namespace: istio-system
      port: 443
      path: /inject
    caBundle: <base64-encoded-CA-cert>  # API Server가 Webhook 서버 인증에 사용
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail   # Webhook 서버 오류 시 요청 거부 (엄격)
  # failurePolicy: Ignore  # Webhook 서버 오류 시 요청 허용 (느슨)
```

```yaml
# Validating Webhook 설정
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: gatekeeper-validating-webhook
webhooks:
- name: check-policy.gatekeeper.sh
  rules:
  - apiGroups: ["*"]
    apiVersions: ["*"]
    resources: ["*"]
    operations: ["CREATE", "UPDATE"]
  clientConfig:
    service:
      name: gatekeeper-webhook-service
      namespace: gatekeeper-system
      port: 443
      path: /v1/admit
    caBundle: <base64-CA>
  failurePolicy: Ignore  # 정책 서버 장애 시 허용
```

### Webhook 서버 구현 구조

```
API Server → HTTPS POST → Webhook 서버
  요청 Body: AdmissionReview (JSON)
  응답 Body: AdmissionReview (JSON)

AdmissionReview 요청:
  {
    "apiVersion": "admission.k8s.io/v1",
    "kind": "AdmissionReview",
    "request": {
      "uid": "abc123",
      "kind": {"group": "", "version": "v1", "kind": "Pod"},
      "object": { ... Pod 스펙 전체 ... },
      "operation": "CREATE",
      "userInfo": {"username": "developer@company.com"}
    }
  }

Mutating 응답 (Sidecar 주입):
  {
    "response": {
      "uid": "abc123",
      "allowed": true,
      "patch": "<base64 JSON Patch>",  # 스펙 수정 내용
      "patchType": "JSONPatch"
    }
  }

Validating 응답 (정책 위반):
  {
    "response": {
      "uid": "abc123",
      "allowed": false,
      "status": {
        "message": "CPU request is required",
        "code": 403
      }
    }
  }
```

### Istio Sidecar 자동 주입

```
전제: Namespace에 istio-injection: enabled 레이블

1. kubectl apply -f deployment.yaml (Pod 생성)

2. API Server → Mutating Webhook 호출
   → istiod의 /inject 엔드포인트
   요청: Pod 스펙

3. istiod Webhook 처리:
   - Pod 스펙에서 컨테이너 목록 확인
   - JSON Patch 생성:
     [
       {"op":"add","path":"/spec/initContainers/-","value": {"name":"istio-init",...}},
       {"op":"add","path":"/spec/containers/-","value": {"name":"istio-proxy",...}},
       {"op":"add","path":"/spec/volumes/-","value": ...},
     ]

4. API Server가 Patch 적용:
   원래 Pod 스펙 → istio-init + 원래 컨테이너들 + istio-proxy 추가

5. 최종 Pod에 Envoy Sidecar 포함
   → iptables 규칙 적용 후 모든 트래픽 Envoy 통과
```

### TLS 인증서와 caBundle

```
Webhook 통신은 반드시 HTTPS:
  이유: API Server가 Webhook 서버의 응답을 신뢰해야 함
       중간자 공격 방지

caBundle 역할:
  Webhook 설정의 clientConfig.caBundle에 CA 인증서를 Base64로 삽입
  API Server가 Webhook 서버 TLS 인증서를 이 CA로 검증
  caBundle이 없거나 잘못되면 API Server가 Webhook 호출 실패

cert-manager로 자동 관리:
  apiVersion: cert-manager.io/v1
  kind: Certificate
  spec:
    dnsNames:
    - istiod.istio-system.svc
    - istiod.istio-system.svc.cluster.local
  → cert-manager가 TLS 인증서 자동 생성/갱신
  → caBundle 자동 주입 (cert-manager의 cainjector)
```

---

## 💻 실전 실험

### 1. Webhook 목록 확인

```bash
# 등록된 Mutating Webhook 확인
kubectl get mutatingwebhookconfigurations
# NAME                          WEBHOOKS
# istio-sidecar-injector        1
# cert-manager-webhook          1
# vpa-webhook-config            1

# 등록된 Validating Webhook 확인
kubectl get validatingwebhookconfigurations
# NAME                                   WEBHOOKS
# gatekeeper-validating-webhook-config   1
# cert-manager-webhook                   1

# 특정 Webhook 상세 확인
kubectl describe mutatingwebhookconfiguration istio-sidecar-injector
```

### 2. Sidecar 주입 확인

```bash
# Namespace에 Istio 주입 레이블 추가
kubectl label namespace default istio-injection=enabled

# 새 파드 생성
kubectl run inject-test --image=nginx

# Sidecar가 주입됐는지 확인
kubectl get pod inject-test -o jsonpath='{.spec.containers[*].name}'
# nginx istio-proxy  ← istio-proxy가 추가됨

# Init Container 확인
kubectl get pod inject-test -o jsonpath='{.spec.initContainers[*].name}'
# istio-init  ← iptables 설정 Init Container
```

### 3. 간단한 Validating Webhook 구현 (개념 확인)

```python
# Webhook 서버 (Python Flask 예시)
from flask import Flask, request, jsonify
import json

app = Flask(__name__)

@app.route('/validate', methods=['POST'])
def validate():
    admission_review = request.json
    pod = admission_review['request']['object']
    
    # 검증: 모든 컨테이너에 resource requests 필수
    containers = pod.get('spec', {}).get('containers', [])
    for container in containers:
        if 'resources' not in container or 'requests' not in container.get('resources', {}):
            return jsonify({
                "apiVersion": "admission.k8s.io/v1",
                "kind": "AdmissionReview",
                "response": {
                    "uid": admission_review['request']['uid'],
                    "allowed": False,
                    "status": {
                        "message": f"Container {container['name']} must have resource requests"
                    }
                }
            })
    
    return jsonify({
        "response": {
            "uid": admission_review['request']['uid'],
            "allowed": True
        }
    })
```

### 4. OPA/Gatekeeper로 정책 강제

```bash
# Gatekeeper 설치
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml

# ConstraintTemplate (정책 스키마 정의)
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requireresources
spec:
  crd:
    spec:
      names:
        kind: RequireResources
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package requireresources
      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not container.resources.requests
        msg := sprintf("Container %v must have resource requests", [container.name])
      }
EOF

# 정책 인스턴스 생성
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireResources
metadata:
  name: all-containers-must-have-requests
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
EOF

# 정책 위반 테스트
kubectl create deployment no-requests --image=nginx
# Error: admission webhook denied the request: Container nginx must have resource requests
```

---

## 📊 Webhook 종류 비교

| 항목 | Mutating Webhook | Validating Webhook |
|-----|---------------|-----------------|
| 호출 순서 | 먼저 (직렬) | 나중 (병렬) |
| 리소스 수정 | ✅ (JSON Patch) | ❌ |
| 요청 거부 | ✅ | ✅ |
| 주요 용도 | Sidecar 주입, 기본값 추가 | 정책 검증, 보안 강제 |
| 대표 예시 | Istio, VPA Admission | OPA Gatekeeper, cert-manager |

---

## ⚖️ 트레이드오프

**failurePolicy: Fail vs Ignore**

`Fail`은 Webhook 서버가 다운되면 해당 리소스의 생성/수정이 모두 실패한다. 보안 정책을 강제할 때는 필수이지만, Webhook 서버가 클러스터 운영의 단일 장애 지점(SPOF)이 된다. `Ignore`는 Webhook 서버 장애 시 요청을 허용하여 클러스터 운영에 영향을 주지 않지만, 보안/정책이 bypass될 수 있다. 중요 보안 Webhook은 `Fail` + 고가용성 Webhook 서버(replicas≥2 + PDB) 조합이 권장된다.

**Webhook 처리 지연**

Webhook 서버 응답 지연이 리소스 생성 속도에 직접 영향을 미친다. Webhook에 100ms가 걸리면 Deployment 10개를 동시에 apply할 때 누적 지연이 발생한다. Webhook 서버 성능 최적화와 타임아웃 설정(`timeoutSeconds: 5` 기본값)이 중요하다.

---

## 📌 핵심 정리

```
API Server 파이프라인:
  인증 → 인가 → Mutating Webhook → 스키마 검증
  → Validating Webhook → etcd 저장

Mutating Webhook:
  JSON Patch로 리소스 스펙 수정
  Istio Sidecar 주입, 기본값 추가
  직렬로 모든 Mutating Webhook 순차 호출

Validating Webhook:
  allowed: true/false 반환
  OPA/Gatekeeper 정책 강제
  병렬로 모든 Validating Webhook 동시 호출

TLS 필수:
  Webhook 서버는 HTTPS 필요
  caBundle로 API Server가 서버 인증서 검증
  cert-manager로 자동 관리 권장

failurePolicy:
  Fail: Webhook 장애 시 요청 거부 (보안 강화)
  Ignore: Webhook 장애 시 요청 허용 (가용성 우선)
```

---

## 🤔 생각해볼 문제

**Q1.** Mutating Webhook이 Pod 스펙을 수정한 후, 그 수정된 내용이 다시 Mutating Webhook의 입력으로 들어오는가?

<details>
<summary>해설 보기</summary>

여러 Mutating Webhook이 등록된 경우, 각 Webhook은 이전 Webhook이 수정한 결과를 입력으로 받는다. 즉, Webhook A가 컨테이너를 추가하면, Webhook B는 A가 추가한 컨테이너를 포함한 스펙을 입력으로 받는다. 단, 같은 Webhook이 수정한 결과를 다시 자신에게 보내지는 않는다(재귀 방지). 이 때문에 Mutating Webhook 간 실행 순서가 중요할 수 있으며, 쿠버네티스는 등록된 순서대로 호출한다.

</details>

**Q2.** `kubectl apply --dry-run=server`는 Webhook을 통과하는가?

<details>
<summary>해설 보기</summary>

그렇다. `--dry-run=server`는 실제 etcd에 저장하지 않지만 API Server 파이프라인(인증, 인가, Admission Webhook 포함)을 모두 통과한다. 따라서 Validating Webhook이 거부하면 dry-run도 실패한다. 이것이 `--dry-run=server`가 `--dry-run=client`보다 더 정확한 이유다. client dry-run은 로컬에서 YAML을 파싱하는 수준이어서 Webhook 정책 위반을 감지하지 못한다. CI/CD에서 배포 전 검증에 `--dry-run=server`를 사용하는 것이 권장된다.

</details>

**Q3.** Webhook 서버가 클러스터 내부의 파드로 실행될 때, 해당 파드가 업그레이드 중이면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

Webhook 서버 파드가 롤링 업데이트 중이면 잠깐 Webhook 서버가 응답하지 않는 순간이 있을 수 있다. `failurePolicy: Fail`이면 이 순간에 모든 리소스 생성/수정이 실패한다. 특히 Webhook 서버 자체를 업데이트할 때 자기 자신의 Webhook을 통과해야 하는 상황이 생기면 교착 상태가 발생할 수 있다. 이를 방지하기 위해 Webhook 설정에 `namespaceSelector`로 Webhook 서버의 네임스페이스를 제외하거나, `objectSelector`로 Webhook 서버 파드를 제외하는 것이 일반적이다.

</details>

---

> ⬅️ 이전: [01. Operator 패턴 — CRD와 Reconciliation Loop](./01-operator-crd.md)  
> ➡️ 다음: [03. Service Mesh — Istio와 Envoy Sidecar](./03-service-mesh-istio.md)
