# ConfigMap과 Secret — 설정 분리의 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 환경변수 주입과 파일 마운트 방식의 동적 갱신 여부가 왜 다른가?
- Secret이 Base64인 것이 암호화가 아닌 이유는 무엇인가?
- etcd에서 Secret을 암호화하려면 어떤 설정이 필요한가?
- Sealed Secret과 External Secrets Operator는 어떻게 GitOps 환경에서 시크릿을 안전하게 관리하는가?
- ConfigMap을 크게 만드는 것보다 여러 개로 나누는 것이 유리한 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"Secret을 Git에 올리면 안 된다"는 것은 알지만, GitOps(코드로 모든 것 관리)를 하면서 Secret 관리가 늘 어렵다. 또한 ConfigMap을 수정했는데 파드에 반영이 안 된다는 문제는 환경변수 vs 파일 마운트 방식을 이해하지 못한 것이다. 운영 환경에서 시크릿 관리 전략은 보안 사고와 직결되므로 올바른 이해가 필수적이다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황 1: ConfigMap을 kubectl apply로 수정했는데
        파드에서 env로 읽으면 여전히 이전 값

원리를 모를 때의 판단:
  "ConfigMap 적용이 안 됐나?" → 다시 apply
  → 여전히 동일

실제 원인:
  envFrom/env으로 주입한 ConfigMap은
  파드 시작 시 환경변수로 복사됨
  → 프로세스 실행 후 원본이 바뀌어도 환경변수는 바뀌지 않음
  → 파드 재시작이 필수

상황 2: Secret을 Base64로 인코딩해서 git에 올림
  "인코딩됐으니 안전하지 않나?"
  → base64는 암호화가 아님 (누구나 디코딩 가능)
  → git 히스토리에 영구적으로 남음
```

---

## ✨ 올바른 접근 (After — 특성을 이해한 설계)

```
설정 갱신 방식 선택:
  재시작 없이 즉시 반영 필요 → 볼륨 마운트 + 앱이 파일 변경 감지
  재시작 허용/간단한 설정   → 환경변수 (envFrom)

Secret 관리 전략:
  개발: Secret YAML을 .gitignore에 추가, 별도 관리
  GitOps 소규모: Sealed Secret (암호화된 YAML을 git에)
  GitOps 대규모: External Secrets Operator (Vault, AWS SM 연동)
  클라우드 네이티브: CSI Secret Store (Vault → Pod 파일 마운트)
```

---

## 🔬 내부 동작 원리

### 환경변수 vs 볼륨 마운트 갱신

```
환경변수 주입 (envFrom/env):
  파드 시작 시 kubelet이 ConfigMap 값을 읽어 프로세스 환경변수로 설정
  → 이후 ConfigMap이 바뀌어도 프로세스 환경변수는 변경되지 않음
  → OS 레벨 제약: 실행 중인 프로세스의 환경변수는 외부에서 변경 불가
  → 파드 재시작 필수

볼륨 마운트 (volumes.configMap):
  kubelet이 syncPeriod(기본 1분)마다 ConfigMap 갱신 체크
  변경 감지 시 symlink 원자적 교체 (rename 시스템콜)
  → 파드 재시작 없이 1~2분 내 파일 내용 자동 변경
  → 단, 앱이 파일 변경을 감지해야 함 (inotify 또는 polling)

확인 방법:
  kubectl exec my-pod -- cat /etc/config/app.properties
  (ConfigMap 수정 1~2분 후 새 내용 확인)
```

### Secret의 본질 — Base64 ≠ 암호화

```
Secret 오브젝트:
  data:
    password: cGFzc3dvcmQ=   ← Base64 인코딩
  
  echo "cGFzc3dvcmQ=" | base64 -d
  # password  ← 즉시 디코딩 가능

Base64가 암호화가 아닌 이유:
  키 없이 누구나 디코딩 가능
  단순히 바이너리 데이터를 텍스트로 표현하는 인코딩
  보안 효과 없음

etcd 저장:
  기본 설정: Secret이 etcd에 평문으로 저장 (Base64 형태)
  etcd에 접근 가능한 사람 → 모든 Secret 조회 가능

etcd 암호화 (EncryptionConfiguration):
  API Server 설정:
    --encryption-provider-config=/etc/kubernetes/enc/config.yaml
  
  config.yaml:
    resources:
    - resources: ["secrets"]
      providers:
      - aescbc:
          keys:
          - name: key1
            secret: <base64-encoded-32-byte-key>
      - identity: {}   # fallback: 암호화 안 됨

  효과:
    etcd에 Secret이 암호화된 형태로 저장
    etcd 직접 접근해도 내용 해독 불가
    API Server를 통해서만 복호화 가능
```

### Sealed Secret — 암호화된 YAML을 Git에

```
동작 원리:
  1. SealedSecrets Controller 클러스터에 설치
     → 비대칭 키쌍 생성 (공개키/개인키)
     → 개인키는 클러스터 내 Secret에 보관
  
  2. 공개키 다운로드
     kubeseal --fetch-cert > pub-cert.pem
  
  3. 개발자가 일반 Secret YAML 작성 후 암호화
     kubeseal --cert pub-cert.pem < secret.yaml > sealed-secret.yaml
  
  4. sealed-secret.yaml을 git에 커밋 (안전)
     (공개키로 암호화 → 클러스터의 개인키 없이는 복호화 불가)
  
  5. ArgoCD/Flux가 sealed-secret.yaml 클러스터에 적용
  
  6. SealedSecrets Controller:
     SealedSecret 오브젝트 감지 → 개인키로 복호화 → Secret 생성

장점: 암호화된 Secret YAML을 git에 안전하게 저장
단점: 클러스터 키 유출 시 모든 Sealed Secret 해독 가능
     키 로테이션 복잡성
```

### External Secrets Operator — 외부 시크릿 매니저 연동

```
지원 백엔드:
  AWS Secrets Manager, AWS SSM Parameter Store
  GCP Secret Manager
  Azure Key Vault
  HashiCorp Vault
  Doppler, 1Password 등

동작 원리:
  1. ESO(External Secrets Operator) 설치
  
  2. SecretStore/ClusterSecretStore 설정
     (백엔드 연결 정보: 어느 AWS SM 리전/계정)
  
  3. ExternalSecret 리소스 생성:
     apiVersion: external-secrets.io/v1beta1
     kind: ExternalSecret
     spec:
       secretStoreRef:
         name: aws-secret-store
       target:
         name: my-k8s-secret  # 생성될 Secret 이름
       data:
       - secretKey: password
         remoteRef:
           key: prod/myapp/db-password  # AWS SM 경로
  
  4. ESO가 AWS SM에서 시크릿 값 조회 → k8s Secret 자동 생성
     주기적으로 동기화 (값 변경 시 자동 반영)

git에 저장: ExternalSecret YAML (Secret 참조만, 값 없음)
실제 값: AWS SM 등 외부 시스템에서 관리

장점: 값이 코드 저장소에 전혀 없음
     외부 시스템의 감사 로그, 접근 제어 활용
단점: 외부 시스템 의존성
     오프라인 환경에서 복잡성 증가
```

---

## 💻 실전 실험

### 1. ConfigMap 갱신 방식 비교

```bash
# ConfigMap 생성
kubectl create configmap update-test --from-literal=message=v1

# envFrom 방식 파드
kubectl run env-pod --image=busybox \
  --env-from=configmap:update-test \
  -- sh -c 'while true; do echo $message; sleep 5; done'

# 볼륨 마운트 방식 파드
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: vol-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do cat /config/message; sleep 5; done']
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    configMap:
      name: update-test
EOF

# ConfigMap 업데이트
kubectl patch configmap update-test --patch='{"data":{"message":"v2"}}'

# 비교: env-pod는 v1 유지, vol-pod는 1~2분 후 v2로 변경
kubectl logs env-pod -f &
kubectl logs vol-pod -f &
```

### 2. Secret Base64 디코딩 확인

```bash
# Secret 생성
kubectl create secret generic my-secret \
  --from-literal=password=supersecret123

# YAML에서 Base64 값 확인
kubectl get secret my-secret -o yaml | grep password
# password: c3VwZXJzZWNyZXQxMjM=

# 즉시 디코딩
kubectl get secret my-secret -o yaml | grep password | \
  awk '{print $2}' | base64 -d
# supersecret123  ← 암호화 아님 확인
```

### 3. Secret 볼륨 마운트와 tmpfs 확인

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-mount-test
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'cat /secrets/password && sleep 3600']
    volumeMounts:
    - name: secret
      mountPath: /secrets
  volumes:
  - name: secret
    secret:
      secretName: my-secret
      defaultMode: 0400
EOF

# tmpfs로 마운트됐는지 확인 (메모리 기반)
kubectl exec secret-mount-test -- cat /proc/mounts | grep secrets
# tmpfs /secrets tmpfs ro,relatime 0 0
```

### 4. immutable ConfigMap/Secret

```yaml
# 자주 변경되지 않는 큰 ConfigMap은 immutable로 설정
# kubelet의 Watch 부하 감소
apiVersion: v1
kind: ConfigMap
metadata:
  name: static-config
immutable: true   # 이후 data 변경 불가 (재생성 필요)
data:
  large-config: |
    ...
```

---

## 📊 설정 관리 방식 비교

| 방식 | 동적 갱신 | git 저장 안전 | 외부 의존성 | 적합 환경 |
|-----|---------|------------|---------|---------|
| ConfigMap 환경변수 | ❌ (재시작 필요) | ✅ | 없음 | 단순 설정 |
| ConfigMap 볼륨 | ✅ (1~2분) | ✅ | 없음 | 동적 설정 필요 |
| Secret (평문 YAML) | ✅ | ❌ (비권장) | 없음 | 개발 환경만 |
| Sealed Secret | 재시작 필요 | ✅ (암호화) | SealedSecrets Controller | GitOps 소규모 |
| External Secrets | ✅ (자동 동기화) | ✅ (값 없음) | AWS SM / Vault 등 | GitOps 대규모 |

---

## ⚖️ 트레이드오프

**ConfigMap 크기 제한**

etcd의 단일 오브젝트 크기 제한은 기본 1.5MB(설정 변경 가능)다. 큰 바이너리나 대용량 설정을 ConfigMap에 넣으면 이 제한에 걸린다. 1MB 이상의 설정은 PVC 볼륨에 저장하거나 Init Container로 외부에서 다운로드하는 것이 나을 수 있다.

**External Secrets Operator의 네트워크 의존성**

ESO는 외부 시스템(AWS SM, Vault 등)에 주기적으로 연결한다. 외부 시스템이 일시적으로 응답하지 않으면 Secret 동기화가 실패하고, 새 파드가 Secret을 생성하지 못할 수 있다. ESO는 마지막으로 성공한 Secret 값을 유지하므로 기존 파드는 영향받지 않는다. 그러나 새 배포나 파드 재시작 시 문제가 될 수 있다.

---

## 📌 핵심 정리

```
ConfigMap 갱신:
  envFrom: 파드 재시작 없이는 갱신 안 됨
  볼륨 마운트: 1~2분 내 자동 갱신 (symlink 원자적 교체)

Secret 보안:
  Base64 = 인코딩 (암호화 아님)
  etcd 기본: 평문 저장 (EncryptionConfiguration 설정 필요)
  파드 마운트: tmpfs (메모리 기반, 디스크 기록 안 됨)

GitOps Secret 전략:
  Sealed Secret: 공개키 암호화 → 암호화된 YAML을 git에
  External Secrets: 값을 외부 시스템에, 참조만 git에

immutable: true:
  변경 불가 설정 → kubelet Watch 부하 감소
  대용량, 자주 안 바뀌는 ConfigMap에 적용
```

---

## 🤔 생각해볼 문제

**Q1.** ConfigMap 볼륨 갱신 후 앱이 새 설정을 바로 사용하지 않는다. 왜 그렇고 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

파일이 변경되어도 앱이 파일을 다시 읽지 않으면 이전 값이 그대로 사용된다. 앱이 처음 시작 시 파일을 읽어 메모리에 올려놓으면, 이후 파일이 바뀌어도 메모리 값은 유지된다. 해결 방법: (1) inotify 파일 변경 감지 후 설정 리로드 (nginx `nginx -s reload`, Spring Boot Actuator refresh 등). (2) sidecar configmap-reloader 컨테이너 사용. (3) polling 방식으로 주기적으로 파일 읽기. 앱 수준에서 핫 리로드를 지원하도록 구현해야 동적 갱신이 의미있다.

</details>

**Q2.** `kubectl get secret my-secret -o yaml`에서 보이는 Secret 데이터를 RBAC으로 숨길 수 있는가?

<details>
<summary>해설 보기</summary>

RBAC으로 `secrets` 리소스의 `get`/`list` 권한을 제거하면 `kubectl get secret`으로 Secret 내용을 조회할 수 없다. 그러나 Namespace 전체 관리자 권한(`admin`)이나 `cluster-admin`이 있으면 조회 가능하다. 또한 RBAC은 API Server 레벨이므로, etcd에 직접 접근하거나 API Server를 우회하는 방법이 있다면 RBAC은 의미없다. 근본적인 시크릿 보호는 etcd 암호화 + API Server의 EncryptionConfiguration 설정이다.

</details>

**Q3.** 하나의 ConfigMap에 모든 설정을 넣는 것과 여러 ConfigMap으로 나누는 것의 차이는?

<details>
<summary>해설 보기</summary>

여러 ConfigMap으로 나누는 것이 유리한 경우가 많다. 첫째, 업데이트 격리가 된다. 한 ConfigMap 변경이 다른 설정에 영향을 주지 않는다. 둘째, RBAC 세분화가 가능하다. 앱 A는 cm-app-a만, 앱 B는 cm-app-b만 읽도록 권한을 설정할 수 있다. 셋째, etcd 오브젝트 크기 제한을 우회할 수 있다. 반면 너무 많이 나누면 관리 복잡성이 증가하므로, 앱별/환경별/기능별로 논리적 단위로 나누는 것이 적절하다.

</details>

---

> ⬅️ 이전: [03. RBAC — 최소 권한 원칙 구현](./03-rbac.md)  
> ➡️ 다음: [05. 모니터링과 로깅 — Prometheus와 Loki 파이프라인](./05-monitoring-logging.md)
