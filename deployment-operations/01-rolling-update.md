# Deployment 롤링 업데이트 — ReplicaSet 교체 알고리즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `maxSurge`와 `maxUnavailable`은 각각 어떤 파드 수를 제어하는가?
- 롤링 업데이트 중 새 ReplicaSet과 기존 ReplicaSet은 어떤 순서로 스케일 조정되는가?
- `kubectl rollout undo`는 내부적으로 어떤 오브젝트를 어떻게 변경하는가?
- `kubectl rollout history`로 어떻게 리비전별 차이를 확인하는가?
- Recreate 전략은 언제 Rolling Update 대신 사용해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

롤링 업데이트는 기본 배포 전략이지만, maxSurge/maxUnavailable 설정에 따라 업데이트 속도와 가용성이 크게 달라진다. "배포 중에 파드가 없는 순간이 있었다"거나, "배포가 너무 느려서 10분이 걸렸다"는 문제는 모두 이 두 값의 설정 문제다. 또한 롤백이 "이전 코드를 다시 빌드하는 것"이 아니라 "기존 ReplicaSet을 다시 스케일업하는 것"임을 알면 롤백이 왜 빠른지 설명할 수 있다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: maxUnavailable=0, maxSurge=0으로 설정
      → kubectl apply 후 배포가 진행되지 않음

원리를 모를 때의 판단:
  "설정이 이상한가?" → 재설정 반복

실제 원인:
  maxUnavailable=0: 기존 파드 하나도 줄일 수 없음
  maxSurge=0:       새 파드 하나도 늘릴 수 없음
  → 교착 상태 (Deadlock): 기존 파드를 줄여야 새 파드를 늘릴 수 있는데
    둘 다 0이면 아무것도 못 함

올바른 이해:
  maxUnavailable=0, maxSurge=1:
    → 먼저 새 파드 1개 추가 후, 기존 파드 1개 제거
    → 항상 원래 replicas 이상 유지 (안전, 느림)

  maxUnavailable=1, maxSurge=0:
    → 먼저 기존 파드 1개 제거 후, 새 파드 1개 추가
    → 순간적으로 replicas-1 상태 (빠름, 일시적 가용성 감소)
```

---

## ✨ 올바른 접근 (After — 알고리즘을 이해한 설정)

```
상황별 maxSurge / maxUnavailable 선택:

안전 우선 (무중단 필수):
  maxSurge: 1
  maxUnavailable: 0
  → 항상 replicas 이상 파드 유지
  → 속도: 파드 하나씩 교체 (느림)
  → 추가 자원 잠깐 필요 (surge 파드)

속도 우선 (일시적 가용성 감소 허용):
  maxSurge: 0
  maxUnavailable: 1
  → 먼저 1개 제거, 그 후 새 파드 추가
  → 속도: 빠름 (추가 자원 불필요)

균형:
  maxSurge: 1
  maxUnavailable: 1
  → 기존 1개 제거 + 새 2개 추가가 동시 진행 가능
  → 속도와 안전성 균형
```

---

## 🔬 내부 동작 원리

### Deployment, ReplicaSet, Pod 관계

```
Deployment (선언)
  "nginx:1.24 파드 3개를 항상 유지하라"
    │
    ├── ReplicaSet v1 (nginx:1.24, replicas=3)
    │     ├── Pod nginx-v1-abc
    │     ├── Pod nginx-v1-def
    │     └── Pod nginx-v1-ghi
    │
    └── (업데이트 후) ReplicaSet v2 (nginx:1.25, replicas=0)
          → 비어있지만 삭제 안 됨 (rollback용으로 보존)
```

### 롤링 업데이트 알고리즘 (maxSurge=1, maxUnavailable=0, replicas=3)

```
초기 상태:
  RS-v1: [Pod-A, Pod-B, Pod-C] (replicas=3)
  RS-v2: [] (replicas=0)
  running=3, unavailable=0, surge=0

  제약: unavailable <= maxUnavailable(0), surge <= maxSurge(1)
        현재: surge=0 < 1 → 새 파드 추가 가능

Step 1: RS-v2 → replicas=1 (새 파드 시작)
  RS-v1: [Pod-A, Pod-B, Pod-C] (running=3)
  RS-v2: [Pod-D(Pending)] (starting)
  surge=1 == maxSurge → 기다림

Step 2: Pod-D Running + Ready (Readiness Probe 통과)
  RS-v1: [Pod-A, Pod-B, Pod-C] (running=3)
  RS-v2: [Pod-D(Running,Ready)]
  총 ready=4, surge=1 → RS-v1에서 1개 제거 가능

Step 3: RS-v1 → replicas=2 (Pod-A 종료)
  RS-v1: [Pod-B, Pod-C]
  RS-v2: [Pod-D]
  ready=3, surge=0 → 새 파드 추가 가능

(반복: Step 1~3를 Pod-B, Pod-C에 대해 반복)

최종 상태:
  RS-v1: [] (replicas=0)
  RS-v2: [Pod-D, Pod-E, Pod-F] (replicas=3)
```

**핵심**: 새 파드가 **Readiness Probe를 통과해야** 기존 파드가 제거된다. Readiness를 구현하지 않으면 컨테이너가 시작되는 순간 Ready로 처리되어 아직 준비 안 된 파드로 트래픽이 들어간다.

### 퍼센트 표기

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # replicas=4이면 ceil(4×0.25)=1
    maxUnavailable: 25%  # replicas=4이면 floor(4×0.25)=1
```

### 롤백 — 이전 ReplicaSet 스케일업

```
kubectl rollout undo deployment/nginx

내부 동작:
  1. Deployment Controller가 revision 히스토리 확인
  2. 직전 ReplicaSet(RS-v1) 찾기
  3. RS-v1.spec.replicas = 3 (스케일업)
  4. RS-v2.spec.replicas = 0 (스케일다운)
  5. 롤링 업데이트와 동일한 알고리즘으로 파드 교체

왜 롤백이 빠른가:
  이미지를 다시 pull하지 않아도 됨
  (노드에 이전 이미지가 캐시됨)
  새 ReplicaSet 생성 없이 기존 RS 재사용
  → 일반 배포와 동일한 속도 (파드 재생성 시간만 소요)
```

```bash
# 리비전 히스토리 확인
kubectl rollout history deployment/nginx
# REVISION  CHANGE-CAUSE
# 1         kubectl apply --filename=nginx-v1.yaml
# 2         kubectl set image deployment/nginx nginx=nginx:1.25

# 특정 리비전 상세 확인
kubectl rollout history deployment/nginx --revision=1

# 특정 리비전으로 롤백
kubectl rollout undo deployment/nginx --to-revision=1

# 롤백 상태 확인
kubectl rollout status deployment/nginx
```

### revisionHistoryLimit — 보존할 ReplicaSet 수

```yaml
spec:
  revisionHistoryLimit: 10  # 기본값: 최근 10개 RS 보존
```

이전 ReplicaSet이 보존되어야 롤백이 가능하다. `0`으로 설정하면 롤백 불가하지만 etcd 공간을 절약한다.

### Recreate 전략

```yaml
spec:
  strategy:
    type: Recreate  # Rolling 아님
```

```
Recreate 동작:
  1. 기존 파드 전체 종료 (replicas → 0)
  2. 완전 종료 확인 후
  3. 새 파드 전체 시작 (0 → replicas)

순간 중단 발생: 기존 파드 종료 ~ 새 파드 Ready 사이 서비스 불가

Recreate 사용 시나리오:
  ReadWriteOnce PV 사용: 두 버전이 동시에 같은 볼륨에 접근하면 충돌
  DB 스키마 변경이 호환 불가: 구버전과 신버전이 공존할 수 없는 경우
  GPU 자원 한계: 노드에 GPU가 1개뿐, 두 파드 동시 실행 불가
```

---

## 💻 실전 실험

### 1. 롤링 업데이트 과정 단계별 관찰

```bash
# Deployment 생성
kubectl create deployment rollout-test --image=nginx:1.24 --replicas=4

# 업데이트 중 파드 상태 실시간 확인
kubectl get pods -l app=rollout-test -w &

# 이미지 업데이트
kubectl set image deployment/rollout-test \
  rollout-test=nginx:1.25

# 롤아웃 진행 상태
kubectl rollout status deployment/rollout-test
# Waiting for deployment "rollout-test" rollout to finish: 1 out of 4 new replicas have been updated...
# Waiting for deployment "rollout-test" rollout to finish: 2 out of 4 new replicas have been updated...
# deployment "rollout-test" successfully rolled out
```

### 2. ReplicaSet 이력 확인

```bash
# RS 목록 (오래된 것 포함)
kubectl get rs -l app=rollout-test
# NAME                    DESIRED   CURRENT   READY   AGE
# rollout-test-abc123     0         0         0       5m   ← 구버전 (replicas=0)
# rollout-test-def456     4         4         4       1m   ← 신버전

# 히스토리 확인
kubectl rollout history deployment/rollout-test
```

### 3. maxSurge/maxUnavailable 차이 비교

```bash
# 빠른 배포 (maxUnavailable=2)
kubectl patch deployment rollout-test -p '
{"spec":{"strategy":{"rollingUpdate":{"maxSurge":0,"maxUnavailable":2}}}}'
time kubectl set image deployment/rollout-test rollout-test=nginx:1.24
kubectl rollout status deployment/rollout-test

# 안전한 배포 (maxUnavailable=0)
kubectl patch deployment rollout-test -p '
{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
time kubectl set image deployment/rollout-test rollout-test=nginx:1.25
kubectl rollout status deployment/rollout-test
```

### 4. 롤백 실습

```bash
# 잘못된 이미지로 배포 (의도적 실패)
kubectl set image deployment/rollout-test rollout-test=nginx:does-not-exist

# 배포 실패 확인
kubectl rollout status deployment/rollout-test
# error: deployment "rollout-test" exceeded its progress deadline

# 이전 버전으로 롤백
kubectl rollout undo deployment/rollout-test

# 롤백 완료 확인
kubectl rollout status deployment/rollout-test
kubectl get pods -l app=rollout-test
```

---

## 📊 배포 전략 비교

| 항목 | RollingUpdate | Recreate |
|-----|-------------|---------|
| 중단 시간 | 없음 (설정 따라) | 있음 (기존 종료~신규 Ready) |
| 자원 사용 | surge 파드만큼 추가 | 동시 파드 없음 |
| 두 버전 공존 | 있음 (롤링 중) | 없음 |
| RWO PVC 호환 | 문제 가능 | 안전 |
| 롤백 속도 | 빠름 (이전 RS 재사용) | 빠름 |
| 적합 워크로드 | 무상태 서비스 | DB 스키마 변경, GPU 워크로드 |

---

## ⚖️ 트레이드오프

**maxSurge=0 vs maxSurge>0**

maxSurge=0으로 설정하면 추가 자원 없이 배포할 수 있어 리소스가 빠듯한 환경에 유리하다. 단, 기존 파드를 먼저 종료해야 하므로 롤링 중 replicas-maxUnavailable 수의 파드만 서비스한다. maxSurge>0이면 추가 자원이 잠깐 필요하지만 항상 replicas 이상의 파드가 Ready 상태를 유지한다.

**revisionHistoryLimit**

히스토리를 많이 보존할수록 롤백 옵션이 늘어나지만 etcd에 빈 ReplicaSet 오브젝트가 쌓인다. 대부분의 경우 최근 3~5개 리비전으로 충분하며, 오래된 빌드로 롤백하는 경우는 드물다.

---

## 📌 핵심 정리

```
롤링 업데이트 알고리즘:
  새 RS 스케일업 ↔ 기존 RS 스케일다운을 교대로 반복
  새 파드가 Ready(Readiness Probe 통과) 후에 기존 파드 제거

maxSurge: 동시에 추가할 수 있는 파드 수 상한
maxUnavailable: 동시에 불가용 상태를 허용할 파드 수

롤백 = 이전 ReplicaSet 스케일업
  etcd에 보존된 기존 RS를 다시 활성화
  새 이미지 빌드/pull 없이 빠른 복구 가능

Recreate: 기존 전체 종료 → 신규 전체 시작
  순간 중단 발생하지만 두 버전 공존이 문제인 경우 사용
```

---

## 🤔 생각해볼 문제

**Q1.** replicas=5, maxSurge=2, maxUnavailable=2일 때 롤링 업데이트 중 동시에 실행 중인 파드 수의 최대/최솟값은?

<details>
<summary>해설 보기</summary>

최댓값: 5(기존) + 2(surge) = 7개. 새 파드 2개가 추가되어 총 7개가 동시에 실행되는 순간이 있다. 최솟값: 5 - 2(unavailable) = 3개. 기존 2개가 종료되고 새 파드가 아직 Ready가 되지 않은 순간에 최소 3개가 Ready 상태다. 실제로는 알고리즘이 maxSurge와 maxUnavailable 제약을 동시에 적용하므로, 정확한 파드 수는 각 단계마다 다르다.

</details>

**Q2.** `kubectl rollout undo` 후 다시 `kubectl rollout undo`를 실행하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

현재 리비전과 직전 리비전 사이를 오간다. 첫 번째 undo로 v2 → v1으로 롤백했다면, 두 번째 undo는 v1 → v2로 돌아간다. 쿠버네티스는 "직전 리비전"을 추적하기 때문이다. 특정 리비전으로 명시적으로 롤백하려면 `kubectl rollout undo deployment/name --to-revision=N`을 사용해야 한다.

</details>

**Q3.** Deployment의 image만 변경하지 않고 ConfigMap을 변경했을 때, 기존 파드가 자동으로 재시작되지 않는 이유는?

<details>
<summary>해설 보기</summary>

Deployment는 Pod Template Spec의 해시값이 변경될 때만 새 ReplicaSet을 생성한다. ConfigMap 내용이 바뀌어도 Deployment의 Pod Template은 `configMapRef.name`만 참조할 뿐 내용 자체를 갖고 있지 않다. 따라서 ConfigMap이 변경되어도 Pod Template 해시가 바뀌지 않아 롤링 업데이트가 트리거되지 않는다. 파드를 재시작하려면 `kubectl rollout restart deployment/name`을 실행하거나, ConfigMap 버전을 Deployment annotation에 기록하는 방법을 사용한다.

</details>

---

> ⬅️ 이전 챕터: [Ch5-05. 클러스터 오토스케일러](../resource-scaling/05-cluster-autoscaler.md)  
> ➡️ 다음: [02. 무중단 배포 — Probe와 Hook의 삼각편대](./02-zero-downtime-deploy.md)
