# StatefulSet — 순서 보장과 안정적 네트워크 ID

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- StatefulSet이 파드를 순서대로 생성/삭제한다는 것이 구체적으로 무슨 의미인가?
- `pod-0.<service>.<namespace>.svc.cluster.local` 형식의 DNS는 어떻게 구현되는가?
- `volumeClaimTemplates`는 Deployment의 PVC 설정과 어떻게 다른가?
- 파드 재생성 후 동일 PVC에 재연결되는 과정은?
- StatefulSet 업데이트 전략(`OnDelete` vs `RollingUpdate`)의 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

PostgreSQL, Kafka, Elasticsearch, Redis Cluster — 상태를 가진 분산 시스템은 파드별로 고유한 역할(Primary/Replica, Broker 번호, 샤드 번호)이 있다. 일반 Deployment로는 이를 구현할 수 없다. 파드가 죽고 재시작될 때 같은 이름, 같은 스토리지, 같은 네트워크 ID가 보장되어야 클러스터가 유지된다. 이것이 StatefulSet이 해결하는 문제다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: PostgreSQL Primary/Replica를 Deployment로 구성
      Primary 파드가 재시작됐는데 다른 이름으로 생성됨

원리를 모를 때의 판단:
  "어차피 서비스로 접근하니 이름이 바뀌어도 되는 거 아닌가?"
  → 설정 유지 시도

실제 문제:
  Replica가 Primary를 "postgres-primary.default.svc..."로 참조 중
  Primary 파드가 재시작되면 랜덤 이름(postgres-abc123)으로 생성
  → Replica가 Primary를 찾지 못해 복제 중단

StatefulSet으로 해결:
  postgres-0은 항상 postgres-0
  postgres-0.postgres-headless.default.svc.cluster.local 고정
  파드 재시작해도 이름/DNS/PVC 동일하게 유지
```

---

## ✨ 올바른 접근 (After — StatefulSet이 적합한 경우)

```
Deployment vs StatefulSet 선택 기준:

Deployment 적합:
  파드가 무상태(stateless) → 어느 파드로 요청이 가도 동일 결과
  웹 서버, API 서버, 마이크로서비스
  수평 확장 시 모든 파드가 동일 역할

StatefulSet 적합:
  각 파드가 고유한 역할 → 순서, ID가 의미있음
  분산 DB (PostgreSQL Primary, MySQL Primary/Replica)
  메시징 (Kafka Broker-0, Broker-1, Broker-2)
  검색 엔진 (Elasticsearch data-0, data-1, data-2)
  캐시 클러스터 (Redis Cluster 노드)
```

---

## 🔬 내부 동작 원리

### 파드 이름과 순서 보장

StatefulSet은 파드에 `<statefulset-name>-<ordinal>` 형식의 고정 이름을 부여한다.

```
StatefulSet 이름: postgres, replicas: 3

생성 순서 (순서 보장):
  postgres-0 생성 → Ready 상태 확인 완료
    ↓ (ready가 되어야 다음 진행)
  postgres-1 생성 → Ready 상태 확인 완료
    ↓
  postgres-2 생성 → Ready 상태 확인 완료

삭제 순서 (역순):
  postgres-2 삭제 → 완전 종료 확인
    ↓ (완전 종료돼야 다음 진행)
  postgres-1 삭제 → 완전 종료 확인
    ↓
  postgres-0 삭제

이 순서 보장으로:
  postgres-0이 Primary로 먼저 시작
  postgres-1, 2가 Replica로 Primary에 접속 후 복제 시작
  스케일다운 시 Replica부터 제거 (Primary 마지막)
```

### Headless Service로 안정적 DNS 구현

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None          # Headless Service
  selector:
    app: postgres
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless  # ← Headless Service 지정
  replicas: 3
  ...
```

```
DNS 이름 생성:
  <pod-name>.<headless-service>.<namespace>.svc.cluster.local

  postgres-0.postgres-headless.default.svc.cluster.local → 10.244.1.5
  postgres-1.postgres-headless.default.svc.cluster.local → 10.244.1.6
  postgres-2.postgres-headless.default.svc.cluster.local → 10.244.2.7

CoreDNS가 각 파드의 PodIP를 위 DNS 이름으로 응답
파드가 재시작돼도 이름은 동일 → DNS는 새 IP로 갱신됨

서비스 DNS (전체 파드 로드밸런싱):
  postgres-headless.default.svc.cluster.local → 모든 파드 IP 목록 반환
```

### volumeClaimTemplates — 파드별 고유 PVC

일반 Deployment의 PVC는 모든 파드가 공유하거나 별도로 수동 생성해야 한다. StatefulSet의 `volumeClaimTemplates`는 파드별로 고유한 PVC를 자동 생성한다.

```yaml
spec:
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

```
자동 생성되는 PVC:
  data-postgres-0    (postgres-0 전용)
  data-postgres-1    (postgres-1 전용)
  data-postgres-2    (postgres-2 전용)

파드 재생성 시 동일 PVC 재연결:
  postgres-0 삭제 → PVC data-postgres-0 유지
  → 새 postgres-0 생성 → volumeClaimTemplates로 data-postgres-0을 찾아 마운트
  → 데이터 그대로 유지

StatefulSet 삭제해도 PVC는 자동 삭제 안 됨 (보호):
  kubectl delete statefulset postgres
  → PVC data-postgres-{0,1,2}는 남아있음
  → 수동 삭제: kubectl delete pvc -l app=postgres
```

### StatefulSet 업데이트 전략

```yaml
spec:
  updateStrategy:
    type: RollingUpdate        # 기본값
    rollingUpdate:
      partition: 0             # 이 ordinal 이상만 업데이트

# 또는
  updateStrategy:
    type: OnDelete             # 수동 삭제 시에만 업데이트
```

```
RollingUpdate 동작:
  replicas=3인 경우:
    postgres-2 업데이트 → Ready 확인
    postgres-1 업데이트 → Ready 확인
    postgres-0 업데이트 → Ready 확인
  (역순으로 하나씩 — Replica 먼저, Primary 나중)

partition 활용 (카나리 업데이트):
  partition: 2
  → ordinal >= 2인 파드만 업데이트 (postgres-2만)
  → postgres-0, postgres-1은 기존 버전 유지
  → 검증 후 partition: 0으로 변경해 전체 업데이트

OnDelete:
  kubectl set image statefulset/postgres ...
  → 아무 일도 안 일어남
  → kubectl delete pod postgres-2 → 삭제 후 새 버전으로 재생성
  → 운영자가 직접 업데이트 타이밍 제어
```

---

## 💻 실전 실험

### 1. StatefulSet 기본 생성

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 1Gi
EOF

# 순서 있는 생성 관찰
kubectl get pods -w
# web-0   0/1  Pending → ContainerCreating → Running
# (web-0 Running 후에야)
# web-1   0/1  Pending → ...
# web-2   0/1  Pending → ...
```

### 2. 파드별 PVC 생성 확인

```bash
kubectl get pvc
# NAME        STATUS  VOLUME   CAPACITY  ACCESSMODES
# www-web-0   Bound   pvc-xxx  1Gi       RWO        ← web-0 전용
# www-web-1   Bound   pvc-yyy  1Gi       RWO        ← web-1 전용
# www-web-2   Bound   pvc-zzz  1Gi       RWO        ← web-2 전용
```

### 3. 안정적 DNS 확인

```bash
kubectl run dns-test --image=busybox --rm -it -- sh

# 파드별 DNS 조회
nslookup web-0.web-headless.default.svc.cluster.local
# Address: 10.244.1.5  (web-0의 IP)

nslookup web-1.web-headless.default.svc.cluster.local
# Address: 10.244.1.6  (web-1의 IP)

# Headless Service → 모든 파드 IP
nslookup web-headless.default.svc.cluster.local
# Address: 10.244.1.5
# Address: 10.244.1.6
# Address: 10.244.2.7
```

### 4. 파드 재시작 후 PVC 재연결 확인

```bash
# web-0에 데이터 기록
kubectl exec web-0 -- sh -c 'echo "pod-0-data" > /usr/share/nginx/html/index.html'

# web-0 강제 삭제 (재시작 시뮬레이션)
kubectl delete pod web-0

# web-0이 재생성될 때까지 대기
kubectl get pod web-0 -w

# 재생성 후 데이터 유지 확인
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
# pod-0-data  ← PVC 재연결로 데이터 유지
```

### 5. partition으로 카나리 업데이트

```bash
# partition=2: web-2만 먼저 업데이트
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'
kubectl set image statefulset/web nginx=nginx:1.25

# web-2만 업데이트됨 확인
kubectl get pods -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
# web-0  nginx:1.24  (기존)
# web-1  nginx:1.24  (기존)
# web-2  nginx:1.25  (업데이트됨)

# 검증 완료 후 전체 업데이트
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

---

## 📊 Deployment vs StatefulSet 비교

| 항목 | Deployment | StatefulSet |
|-----|-----------|------------|
| 파드 이름 | 랜덤 (xxx-abc123) | 고정 (name-0, name-1) |
| 생성/삭제 순서 | 병렬, 순서 없음 | 순서 보장 (0→1→2 / 역순) |
| 네트워크 ID | ClusterIP Service | Headless Service + 파드별 DNS |
| 스토리지 | 공유 PVC 또는 없음 | volumeClaimTemplates로 파드별 PVC |
| 업데이트 방식 | 병렬 롤링 | 역순 롤링 또는 OnDelete |
| 주요 사용 사례 | 무상태 서비스 | DB, 메시징, 분산 캐시 |

---

## ⚖️ 트레이드오프

**StatefulSet의 느린 업데이트**

파드를 하나씩 순서대로 업데이트하므로, 3개 파드의 업데이트가 각 파드 기동 시간의 3배가 걸린다. Spring Boot처럼 기동이 느린 애플리케이션에서는 업데이트에 수분이 소요된다. 이를 수용할 수 없다면 OnDelete 전략으로 업데이트 타이밍을 직접 제어하거나, partition을 활용해 점진적으로 업데이트한다.

**StatefulSet PVC의 수동 관리 필요성**

StatefulSet을 삭제해도 PVC는 자동으로 삭제되지 않는다. 이는 데이터 보호를 위한 의도적 설계이지만, 스테이징 환경에서 정리할 때 PVC를 별도로 삭제해야 하는 불편함이 있다. Kubernetes 1.27부터 `persistentVolumeClaimRetentionPolicy`로 StatefulSet 삭제/스케일다운 시 PVC 자동 삭제 정책을 설정할 수 있다.

---

## 📌 핵심 정리

```
StatefulSet이 보장하는 세 가지:
  1. 안정적 이름: <name>-0, <name>-1 (재시작해도 동일)
  2. 안정적 DNS: <pod>.<headless-svc>.<ns>.svc.cluster.local
  3. 안정적 스토리지: volumeClaimTemplates로 파드별 PVC

생성/삭제 순서:
  생성: 0 → 1 → 2 (오름차순, 각 파드 Ready 후 다음 진행)
  삭제: 2 → 1 → 0 (내림차순, 각 파드 종료 후 다음 진행)

volumeClaimTemplates:
  자동 생성: data-<name>-0, data-<name>-1 ...
  StatefulSet 삭제 후에도 PVC 유지 (데이터 보호)
  파드 재생성 시 같은 PVC 자동 재연결

업데이트:
  RollingUpdate: 역순(2→1→0), partition으로 카나리 가능
  OnDelete: 직접 pod 삭제 시에만 업데이트 (수동 제어)
```

---

## 🤔 생각해볼 문제

**Q1.** StatefulSet replicas를 3에서 1로 줄이면 어떤 순서로 파드가 삭제되고, PVC는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

역순으로 삭제된다. postgres-2가 먼저 삭제되고, 완전 종료 후 postgres-1이 삭제된다. postgres-0은 남는다. 삭제된 파드들의 PVC(data-postgres-1, data-postgres-2)는 자동으로 삭제되지 않고 Released 상태로 남는다. 이는 의도적 설계로, 나중에 replicas를 다시 3으로 늘리면 기존 PVC가 재사용된다. 완전히 정리하려면 `kubectl delete pvc data-postgres-1 data-postgres-2`를 수동으로 실행해야 한다.

</details>

**Q2.** StatefulSet의 Primary 파드(postgres-0)가 죽었을 때, 애플리케이션이 자동으로 failover되는가?

<details>
<summary>해설 보기</summary>

쿠버네티스 자체는 failover를 처리하지 않는다. StatefulSet은 postgres-0을 재시작하는 것만 보장한다. Primary failover는 애플리케이션 레벨에서 구현해야 한다. PostgreSQL의 경우 Patroni 같은 HA 솔루션이 필요하다. Patroni는 DCS(etcd/Consul/ZooKeeper)를 통해 Primary 선출을 처리하고, Replica를 새 Primary로 승격시킨다. StatefulSet은 단지 파드의 이름과 스토리지 고정을 보장할 뿐, 클러스터 관리 로직은 별도 레이어가 담당한다.

</details>

**Q3.** StatefulSet에서 initContainers를 사용해 각 파드가 자신의 ordinal을 알 수 있는가?

<details>
<summary>해설 보기</summary>

그렇다. 파드의 이름이 `<statefulset>-<ordinal>`이므로, `metadata.name`을 환경변수로 주입하면 ordinal을 파싱할 수 있다. `env: - name: POD_NAME; valueFrom.fieldRef.fieldPath: metadata.name`으로 파드 이름을 환경변수로 넘기고, init container 또는 애플리케이션에서 `${POD_NAME##*-}`으로 ordinal을 추출한다. 이를 통해 postgres-0은 Primary 역할, postgres-1+는 Replica 역할을 스스로 판단해 초기화할 수 있다.

</details>

---

> ⬅️ 이전: [03. CSI — 스토리지 드라이버 표준 인터페이스](./03-csi.md)  
> ➡️ 다음: [05. 스토리지 성능 고려사항](./05-storage-performance.md)
