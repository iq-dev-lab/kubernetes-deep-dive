# Operator 패턴 — CRD와 Reconciliation Loop

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CRD는 어떻게 쿠버네티스 API를 도메인 특화 리소스로 확장하는가?
- Reconciliation Loop란 무엇이고, 왜 이벤트 기반이 아닌 상태 기반으로 구현해야 하는가?
- controller-runtime 라이브러리를 사용한 Operator의 기본 구조는 어떻게 생겼는가?
- Finalizer는 언제, 왜 사용하는가?
- 실제 Postgres Operator는 무엇을 어떻게 자동화하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"DB를 쿠버네티스에서 안전하게 운영하려면 Operator를 써야 한다"는 말을 자주 듣는다. 그 이유는 Operator가 사람이 해야 하는 운영 지식(장애 감지, 페일오버, 백업, 스케일링)을 쿠버네티스 컨트롤러로 코드화하기 때문이다. CRD와 Reconciliation Loop를 이해하면 Operator가 어떻게 동작하는지, 무슨 문제를 해결하는지 명확히 알 수 있다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: CRD를 생성했는데 Custom Resource를 만들어도 아무 일도 안 일어남

원리를 모를 때의 판단:
  "CRD 설정이 잘못됐나?" → CRD 재작성
  → 동일 증상

실제 원인:
  CRD = API 스키마 정의 (어떤 필드가 있는가)
  Custom Controller = 실제 동작 구현 (CR 감지 후 무엇을 할지)
  
  CRD만 있고 Controller가 없으면
  CR 오브젝트는 etcd에 저장되지만 아무 동작도 하지 않음
  
  Operator = CRD + Custom Controller (둘의 조합)
```

---

## ✨ 올바른 접근 (After — Operator 구성요소 이해)

```
Operator를 만들기 위한 세 가지:

1. CRD (Custom Resource Definition)
   API 스키마 정의: 어떤 필드가 있는가, 유효성 규칙
   
2. Custom Resource (CR)
   CRD로 정의된 타입의 인스턴스
   kubectl apply -f my-database.yaml

3. Custom Controller
   CR의 Desired State를 감지
   → 외부 시스템(클라우드, DB) 상태와 비교
   → Reconcile (차이를 해소하는 작업 수행)
   → status 업데이트

대표 Operator 예시:
  Postgres Operator (Zalando): PostgreSQL 클러스터 관리
  Kafka Operator (Strimzi): Kafka 클러스터 + 토픽 관리
  Prometheus Operator: Prometheus + AlertManager 관리
  CertManager: TLS 인증서 자동 발급/갱신
```

---

## 🔬 내부 동작 원리

### CRD 구조

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.database.mycompany.io
spec:
  group: database.mycompany.io
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: ["replicas", "version"]
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 9
              version:
                type: string
                enum: ["14", "15", "16"]
              storageSize:
                type: string
                default: "10Gi"
          status:
            type: object
            properties:
              phase:
                type: string    # Running / Degraded / Creating
              primaryPod:
                type: string
  scope: Namespaced
  names:
    plural: postgresclusters
    singular: postgrescluster
    kind: PostgresCluster
    shortNames: ["pg"]
```

```bash
# CRD 등록 후 kubectl로 사용 가능
kubectl get postgresclusters -A
kubectl get pg -A       # shortName 사용
```

### Custom Resource (CR)

```yaml
apiVersion: database.mycompany.io/v1
kind: PostgresCluster
metadata:
  name: prod-db
  namespace: production
spec:
  replicas: 3
  version: "15"
  storageSize: "100Gi"
```

### Reconciliation Loop — 핵심 설계 원칙

```
이벤트 기반 (잘못된 방식):
  CR 생성 이벤트 → StatefulSet 생성
  CR 수정 이벤트 → StatefulSet 업데이트
  
  문제:
    이벤트 유실 시 상태 불일치
    Controller 재시작 시 이벤트 재생 불가
    여러 이벤트가 동시에 오면 순서 보장 어려움

상태 기반 Reconciliation (올바른 방식):
  어떤 이벤트가 와도 항상 같은 동작:
    1. CR의 현재 spec 조회 (Desired State)
    2. 실제 쿠버네티스 리소스 상태 조회 (Actual State)
    3. 차이 계산 및 해소

  멱등성(Idempotency):
    같은 입력 → 항상 같은 결과
    여러 번 실행해도 안전
    
  Reconcile 함수:
    func (r *PostgresClusterReconciler) Reconcile(ctx context.Context, req Request) (Result, error) {
      // 1. CR 조회
      cluster := &PostgresCluster{}
      r.Get(ctx, req.NamespacedName, cluster)
      
      // 2. StatefulSet 상태 확인
      sts := &StatefulSet{}
      err := r.Get(ctx, types.NamespacedName{Name: cluster.Name, Namespace: cluster.Namespace}, sts)
      
      if errors.IsNotFound(err) {
        // StatefulSet 없음 → 생성
        return r.createStatefulSet(ctx, cluster)
      }
      
      // 3. replicas 불일치 → 업데이트
      if *sts.Spec.Replicas != int32(cluster.Spec.Replicas) {
        sts.Spec.Replicas = &cluster.Spec.Replicas
        r.Update(ctx, sts)
      }
      
      // 4. status 업데이트
      cluster.Status.Phase = "Running"
      r.Status().Update(ctx, cluster)
      
      return Result{RequeueAfter: 30 * time.Second}, nil  // 30초 후 재확인
    }
```

### Finalizer — 삭제 전 정리 작업

```
문제: CR 삭제 시 연관 외부 리소스(AWS RDS, Vault 시크릿)를 정리해야 함
     kubectl delete postgresql → CR만 삭제되고 RDS는 남음

해결: Finalizer

  1. CR 생성 시 Finalizer 추가:
     metadata.finalizers: ["database.mycompany.io/cleanup"]
  
  2. kubectl delete postgresql 실행:
     → CR에 DeletionTimestamp 설정 (즉시 삭제 안 됨)
     → Finalizer가 있으면 삭제 보류
  
  3. Controller가 DeletionTimestamp 감지:
     → 외부 리소스 정리 (AWS RDS 삭제, Vault 정리)
     → 완료 후 Finalizer 제거:
        controllerutil.RemoveFinalizer(cluster, "database.mycompany.io/cleanup")
        r.Update(ctx, cluster)
  
  4. Finalizer가 없으면 API Server가 CR 최종 삭제
```

---

## 💻 실전 실험

### 1. 간단한 CRD 생성 및 CR 사용

```bash
# CRD 등록
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.mycompany.io
spec:
  group: mycompany.io
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              url:
                type: string
              replicas:
                type: integer
                default: 1
  scope: Namespaced
  names:
    plural: websites
    singular: website
    kind: Website
EOF

# CR 생성
cat <<EOF | kubectl apply -f -
apiVersion: mycompany.io/v1
kind: Website
metadata:
  name: my-site
spec:
  url: https://mysite.com
  replicas: 3
EOF

kubectl get websites
kubectl get website my-site -o yaml
```

### 2. Controller 동작 시뮬레이션 (kubebuilder 구조)

```bash
# kubebuilder로 Operator 프로젝트 생성 (개념 파악용)
# kubebuilder init --domain mycompany.io --repo github.com/mycompany/website-operator
# kubebuilder create api --group mycompany --version v1 --kind Website

# 생성된 controller 파일 구조:
# internal/controller/website_controller.go
#   func (r *WebsiteReconciler) Reconcile(...) (ctrl.Result, error)
#   SetupWithManager: 어떤 오브젝트 Watch할지 설정
```

### 3. 실제 Operator 설치 - cert-manager

```bash
# cert-manager 설치 (가장 유명한 Operator 중 하나)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# CRD 확인
kubectl get crd | grep cert-manager
# certificaterequests.cert-manager.io
# certificates.cert-manager.io
# clusterissuers.cert-manager.io
# issuers.cert-manager.io

# ClusterIssuer CR 생성 (Let's Encrypt)
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Certificate CR 생성 → cert-manager가 자동으로 인증서 발급
kubectl get certificate -A -w
```

---

## 📊 Operator 성숙도 레벨 (Operator Framework)

| 레벨 | 기능 | 예시 |
|-----|-----|-----|
| Level 1 | 자동 설치/업그레이드 | Helm Chart 수준 |
| Level 2 | 설정 관리 (설정 변경 자동 반영) | 기본 Operator |
| Level 3 | 자동 복구 (Primary 장애 시 Failover) | 중급 Operator |
| Level 4 | 자동 스케일링 (부하 기반 노드 추가) | 고급 Operator |
| Level 5 | 자동 튜닝 (사용 패턴 분석, 파라미터 최적화) | 전문 Operator |

---

## ⚖️ 트레이드오프

**Operator 직접 개발 vs 기존 Operator 사용**

PostgreSQL을 위한 Zalando Postgres Operator나 Kafka를 위한 Strimzi는 수년간 검증된 성숙한 Operator다. 직접 만드는 것보다 이런 기존 Operator를 사용하는 것이 대부분의 경우 더 안전하고 빠르다. 그러나 조직 내 고유한 비즈니스 로직(사내 배포 승인 프로세스, 독자적인 DR 정책)이 있다면 직접 Operator를 개발해야 한다.

**Reconcile 빈도**

Reconcile이 너무 자주 실행되면 API Server 부하가 증가한다. `RequeueAfter: 30s` 같이 주기적 재확인을 설정하고, Watch를 통한 이벤트 기반 트리거를 함께 사용하는 것이 일반적이다. 중요한 변경은 Watch로 즉시 감지하고, 외부 시스템 상태 동기화는 주기적으로 확인한다.

---

## 📌 핵심 정리

```
Operator = CRD + Custom Controller
  CRD:        API 스키마 정의 (어떤 필드가 있는가)
  Controller: Reconciliation Loop (Desired vs Actual → 차이 해소)

Reconciliation Loop 핵심:
  멱등성: 여러 번 실행해도 동일한 결과
  상태 기반: 이벤트 유실해도 상태에서 항상 복구 가능
  status 업데이트: 현재 상태를 CR status에 기록

Finalizer:
  CR 삭제 전 외부 리소스 정리
  DeletionTimestamp → 정리 완료 → Finalizer 제거 → 실제 삭제

유명 Operator:
  cert-manager, Prometheus Operator, Zalando Postgres Operator, Strimzi
```

---

## 🤔 생각해볼 문제

**Q1.** Reconcile 함수가 에러를 반환하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

controller-runtime이 자동으로 지수 백오프(exponential backoff)로 Reconcile을 재시도한다. 처음에는 즉시 재시도, 이후 1초, 2초, 4초... 최대 30초 간격으로 늘어난다. 이 재시도 메커니즘 덕분에 일시적인 API Server 오류나 네트워크 오류가 있어도 결국 Reconcile이 성공한다. 영구적인 오류(잘못된 스펙 등)는 status에 에러를 기록하고 계속 재시도한다.

</details>

**Q2.** CR의 status 필드를 업데이트할 때 일반 `Update()`가 아닌 `Status().Update()`를 사용하는 이유는?

<details>
<summary>해설 보기</summary>

CRD에 status subresource가 활성화된 경우, spec과 status는 별도의 API 엔드포인트로 분리된다. `Update()`는 spec 필드를 업데이트하고, `Status().Update()`는 status 필드를 업데이트한다. 이렇게 분리되면 사용자(kubectl apply)와 Controller(Reconcile) 간의 충돌을 방지한다. 사용자는 spec을 변경하고, Controller는 status를 업데이트하는 역할 분리가 명확해진다. `Update()`로 status를 변경하면 spec 업데이트와 충돌하거나, status가 spec과 함께 덮어써질 수 있다.

</details>

**Q3.** 여러 종류의 CR(PostgresCluster, PostgresBackup)이 있을 때 하나의 Controller에서 관리하는 것과 각각 별도 Controller를 두는 것의 장단점은?

<details>
<summary>해설 보기</summary>

단일 Controller는 코드가 단순하지만, 하나의 CR 타입에 버그가 생기면 다른 CR 처리에도 영향을 미칠 수 있다. 별도 Controller는 각 CR 타입의 로직을 독립적으로 개발·테스트·배포할 수 있다. 일반적으로 관련된 리소스를 하나의 Operator Binary(하나의 Pod)에서 여러 Reconciler로 관리하는 방식이 많이 사용된다. controller-runtime의 SetupWithManager를 통해 하나의 Manager에 여러 Reconciler를 등록할 수 있다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Admission Webhook — 리소스 생성/수정 인터셉트 ➡️](./02-admission-webhook.md)**

</div>
