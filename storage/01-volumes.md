# Volume과 파드 — emptyDir부터 configMap까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- emptyDir, hostPath, configMap, secret은 각각 어떻게 마운트되는가?
- 컨테이너가 재시작될 때와 파드가 재시작될 때 데이터 유지 여부가 왜 다른가?
- emptyDir의 `medium: Memory`는 내부적으로 무엇인가?
- kubelet은 볼륨을 어떤 순서로 파드에 마운트하는가?
- hostPath가 보안 위험으로 간주되는 이유는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"파드를 재시작했더니 데이터가 사라졌다"는 장애의 원인 대부분이 볼륨 종류를 잘못 이해한 것이다. emptyDir은 파드 수명과 함께하므로, 파드가 재스케줄링되면 사라진다. configMap/secret이 업데이트됐는데 파드 내에서 반영이 안 된다고 할 때, 마운트 방식(볼륨 vs 환경변수)에 따라 동적 반영 여부가 다르다는 것을 알아야 한다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황 1: 컨테이너가 OOM으로 재시작됐는데 /data 디렉토리 데이터가 사라짐

원리를 모를 때의 판단:
  "컨테이너가 재시작됐으니 데이터는 당연히 사라지는 것 아닌가?"
  → 그냥 방치, 재시작마다 초기화 감수

실제 동작:
  emptyDir는 파드 수명에 묶여있음
  컨테이너 재시작 ≠ 파드 재시작
  컨테이너만 재시작(OOM kill 등)되면 emptyDir 데이터는 유지됨
  파드가 다른 노드로 재스케줄링될 때만 emptyDir 삭제

상황 2: configMap을 수정했는데 파드 내부에서 바뀐 값이 안 보임

원리를 모를 때의 판단:
  "파드를 재시작해야 반영되는 건가?" → 재시작
  
실제 동작:
  볼륨으로 마운트한 configMap → kubelet이 약 1분 내 자동 갱신
  envFrom/env로 주입한 configMap → 파드 재시작 없이는 절대 반영 안 됨
```

---

## ✨ 올바른 접근 (After — 볼륨 종류를 이해한 설계)

```
볼륨 수명 기준으로 선택:

  파드 수명과 동일 (파드 삭제 시 삭제)
    emptyDir        → 컨테이너 간 공유, 캐시, 임시 작업 디렉토리

  노드 수명과 동일 (노드가 유지되는 한 유지)
    hostPath        → 로그 수집(DaemonSet), 노드 수준 설정 접근

  클러스터 오브젝트 (쿠버네티스 오브젝트 수명)
    configMap       → 설정 파일
    secret          → 인증서, 패스워드
    projected       → 여러 소스를 하나의 경로로 합성

  영구 스토리지 (PVC로 관리)
    PersistentVolumeClaim → DB 데이터, 사용자 파일
    → [02. PV와 PVC](./02-pv-pvc-storageclass.md) 참고
```

---

## 🔬 내부 동작 원리

### kubelet의 볼륨 마운트 순서

파드가 노드에 배정되면 kubelet은 컨테이너 실행 전에 볼륨을 순서대로 준비한다.

```
kubelet 볼륨 처리 순서:

  1. Volume Manager가 볼륨 준비 시작
     ConfigMap/Secret → API Server에서 데이터 조회
     PVC → CSI 드라이버 호출 (Attach/Mount)
     emptyDir → 로컬 디렉토리 생성
     hostPath → 접근 권한 확인

  2. 준비된 볼륨을 파드 전용 디렉토리에 배치
     /var/lib/kubelet/pods/<pod-uid>/volumes/<volume-type>/<volume-name>/

  3. containerd에 컨테이너 생성 요청 시
     config.json의 mounts[] 배열에 볼륨 경로 포함
     → 컨테이너 시작 시 bind mount로 연결

  4. 모든 Init Container와 메인 컨테이너가 같은 볼륨 경로를 바라봄
```

### emptyDir — 파드 수명의 임시 스토리지

```yaml
volumes:
- name: cache
  emptyDir: {}          # 기본: 노드 디스크
  # emptyDir:
  #   medium: Memory    # tmpfs (메모리 기반)
  #   sizeLimit: 256Mi  # 크기 제한
```

```
emptyDir 생성 위치:
  기본: /var/lib/kubelet/pods/<uid>/volumes/kubernetes.io~empty-dir/cache/
  → 노드의 실제 디스크 공간 사용
  → 파드 삭제 시 kubelet이 rm -rf로 정리

medium: Memory (tmpfs):
  실제 마운트: mount -t tmpfs tmpfs /var/lib/kubelet/pods/.../cache/
  → 메모리에 저장 → 노드 메모리 자원에서 차감
  → 재시작(컨테이너 재시작 포함)해도 지워지지 않으나
     파드 종료 시 즉시 사라짐
  → 고속 읽기/쓰기 필요 시 (공유 메모리 IPC, 고속 캐시)

컨테이너 재시작 vs 파드 재시작:
  컨테이너만 재시작 (OOM kill, Liveness 실패):
    emptyDir → 유지 (파드 uid 디렉토리가 살아있음)
  
  파드 재스케줄링 (노드 변경):
    emptyDir → 삭제 (새 노드의 새 파드 uid 디렉토리)
```

### hostPath — 노드 파일시스템 직접 접근

```yaml
volumes:
- name: node-logs
  hostPath:
    path: /var/log           # 노드의 실제 경로
    type: Directory          # Directory / File / DirectoryOrCreate 등
```

```
동작 원리:
  컨테이너의 /logs → 노드의 /var/log bind mount
  파드가 삭제되어도 노드의 /var/log는 그대로 유지

주요 사용 사례:
  DaemonSet으로 각 노드 로그 수집 (Fluentd, Filebeat)
  → /var/log/pods를 hostPath로 마운트해 모든 파드 로그 수집

보안 위험:
  악의적인 파드가 hostPath: path: / 로 노드 전체 파일시스템 접근 가능
  → 컨테이너 탈출 취약점과 결합 시 노드 완전 장악
  → PodSecurityAdmission으로 hostPath 사용 제한 권장
  → 최소 path를 구체적으로 지정하고, type을 명시해야 함
```

### configMap 마운트 — 동적 갱신의 원리

configMap을 볼륨으로 마운트하면 kubelet이 주기적으로 최신 내용을 파드에 반영한다.

```yaml
volumes:
- name: app-config
  configMap:
    name: my-config
    items:                    # 특정 키만 마운트 (미지정 시 전체)
    - key: app.properties
      path: app.properties

containers:
- volumeMounts:
  - name: app-config
    mountPath: /etc/app       # /etc/app/app.properties 파일로 마운트
    readOnly: true
```

```
마운트 구현 방식 (symlink 기반):
  /var/lib/kubelet/pods/<uid>/volumes/kubernetes.io~configmap/app-config/
    app.properties            ← 실제 파일 아님, symlink
    ..2024_01_01_00_00_00.123456789/  ← 실제 내용 담긴 디렉토리 (타임스탬프)
      app.properties          ← 실제 파일

갱신 동작:
  1. ConfigMap 업데이트 (kubectl apply)
  2. kubelet이 ConfigMap 변화 감지 (syncPeriod: 기본 1분)
  3. 새 타임스탬프 디렉토리 생성 후 새 내용 기록
  4. symlink 원자적 교체 (rename 시스템콜)
  → 파드 재시작 없이 약 1~2분 내 자동 반영

envFrom/env 방식은 갱신 안 됨:
  환경변수는 프로세스 시작 시 복사 → 이후 원본 변경 무관
  → 파드 재시작 없이는 절대 반영 불가
```

### secret 마운트 — tmpfs 기반 보안

```yaml
volumes:
- name: tls-certs
  secret:
    secretName: my-tls-cert
    defaultMode: 0400         # 소유자 읽기 전용
```

```
secret 볼륨의 특이점:
  tmpfs로 마운트 (메모리 기반)
  → 노드 디스크에 평문으로 기록되지 않음
  → 파드 종료 시 즉시 메모리에서 제거
  → /proc/mounts에서 확인 가능: tmpfs /run/secrets/... tmpfs

주의: etcd에는 암호화되지 않은 채 저장됨 (별도 설정 필요)
      API Server의 --encryption-provider-config 설정으로 etcd 암호화
```

---

## 💻 실전 실험

### 1. emptyDir — 컨테이너 재시작 후 데이터 유지 확인

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-test
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "data-$(date)" > /shared/data.txt && sleep 3600']
    volumeMounts:
    - name: shared
      mountPath: /shared
    livenessProbe:
      exec:
        command: ['false']   # 항상 실패 → 컨테이너 재시작 유발
      periodSeconds: 30
  volumes:
  - name: shared
    emptyDir: {}
EOF

# 최초 데이터 확인
kubectl exec emptydir-test -- cat /shared/data.txt
# data-Mon Jan  1 00:00:00 UTC 2024

# 컨테이너 재시작 후 데이터 확인 (약 30초 후)
kubectl get pod emptydir-test -w   # RESTARTS 증가 기다림
kubectl exec emptydir-test -- cat /shared/data.txt
# data-Mon Jan  1 00:00:00 UTC 2024  ← 동일! emptyDir 유지
```

### 2. configMap 볼륨 동적 갱신 확인

```bash
# ConfigMap 생성
kubectl create configmap my-config --from-literal=message="v1"

# 볼륨으로 마운트하는 파드
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cm-volume-test
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do cat /etc/config/message; sleep 5; done']
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: my-config
EOF

kubectl logs cm-volume-test -f &

# ConfigMap 업데이트
kubectl patch configmap my-config --patch='{"data":{"message":"v2"}}'

# 약 1~2분 후 로그에서 v2 출력 확인 (파드 재시작 없이!)
```

### 3. hostPath 마운트 확인

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-test
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'ls /node-logs && sleep 3600']
    volumeMounts:
    - name: logs
      mountPath: /node-logs
  volumes:
  - name: logs
    hostPath:
      path: /var/log
      type: Directory
EOF

# 파드 안에서 노드의 /var/log 내용 접근
kubectl exec hostpath-test -- ls /node-logs
```

### 4. 볼륨 마운트 경로 노드에서 직접 확인

```bash
POD_UID=$(kubectl get pod emptydir-test -o jsonpath='{.metadata.uid}')
docker exec -it kind-worker bash

# emptyDir 실제 위치
ls /var/lib/kubelet/pods/$POD_UID/volumes/kubernetes.io~empty-dir/

# configMap 심링크 구조 확인
ls -la /var/lib/kubelet/pods/*/volumes/kubernetes.io~configmap/*/
```

### 5. secret tmpfs 마운트 확인

```bash
kubectl create secret generic my-secret --from-literal=password=s3cr3t

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-test
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'cat /secrets/password && sleep 3600']
    volumeMounts:
    - name: secret-vol
      mountPath: /secrets
  volumes:
  - name: secret-vol
    secret:
      secretName: my-secret
      defaultMode: 0400
EOF

# secret이 tmpfs에 마운트됐는지 확인
kubectl exec secret-test -- cat /proc/mounts | grep secret
# tmpfs /secrets tmpfs ro,...  ← tmpfs 확인
```

---

## 📊 볼륨 종류별 비교

| 볼륨 종류 | 수명 | 컨테이너 재시작 후 | 파드 재스케줄 후 | 주요 용도 |
|---------|-----|---------------|--------------|---------|
| emptyDir (disk) | 파드 수명 | 유지 | 삭제 | 컨테이너 간 공유, 임시 캐시 |
| emptyDir (memory) | 파드 수명 | 삭제 | 삭제 | 고속 공유 메모리, IPC |
| hostPath | 노드 수명 | 유지 | 노드 따라 다름 | 로그 수집(DaemonSet) |
| configMap | 오브젝트 수명 | 유지 (동적 갱신) | 유지 | 설정 파일 |
| secret | 오브젝트 수명 | 유지 (tmpfs) | 유지 | 인증서, 패스워드 |
| PVC | 볼륨 정책 | 유지 | 유지 | DB, 영구 데이터 |

---

## ⚖️ 트레이드오프

**configMap 볼륨 vs 환경변수**

볼륨 마운트 방식은 파드 재시작 없이 동적 갱신이 가능하지만, 파일 기반이므로 애플리케이션이 파일 변경을 감지하는 로직이 필요하다. 환경변수 방식은 파드 재시작 없이는 갱신이 불가능하지만, 애플리케이션 코드 변경이 불필요하고 사용이 단순하다. 동적 설정 변경이 필요하다면 볼륨 마운트 + inotify 감지 또는 configmap-reload 사이드카 패턴을 사용한다.

**hostPath의 필요성과 보안 균형**

DaemonSet 기반 로그 수집이나 모니터링 에이전트는 hostPath 없이 구현하기 어렵다. 그러나 hostPath 경로를 최대한 좁히고(`/var/log/pods`처럼 구체적으로), `readOnly: true`를 설정하며, PodSecurityAdmission으로 허용 경로를 제한하는 것이 필요하다.

---

## 📌 핵심 정리

```
볼륨 수명:
  emptyDir → 파드 수명 (컨테이너 재시작 시 유지, 파드 종료 시 삭제)
  hostPath → 노드에 영속 (파드와 무관)
  configMap/secret → 쿠버네티스 오브젝트 수명

kubelet 볼륨 처리:
  컨테이너 실행 전 볼륨 준비
  /var/lib/kubelet/pods/<uid>/volumes/ 에 배치
  config.json mounts[]에 bind mount로 전달

configMap 동적 갱신:
  볼륨 마운트: 심링크 원자적 교체 → 약 1~2분 내 자동 반영
  환경변수: 파드 재시작 없이 절대 반영 안 됨

secret = tmpfs:
  메모리 기반 마운트 → 노드 디스크에 평문 기록 안 됨
  파드 종료 시 메모리에서 즉시 제거
```

---

## 🤔 생각해볼 문제

**Q1.** Init Container와 메인 컨테이너가 emptyDir를 공유할 때, Init Container가 완료되면 기록한 파일이 메인 컨테이너에서 보이는가?

<details>
<summary>해설 보기</summary>

그렇다. emptyDir는 파드 수명에 묶여 있고, Init Container와 메인 컨테이너는 같은 파드 내에서 동일한 emptyDir를 마운트한다. Init Container가 파일을 기록하고 종료해도 파드 uid 디렉토리와 emptyDir 데이터는 유지되므로, 메인 컨테이너가 시작될 때 해당 파일을 읽을 수 있다. 이것이 Init Container 패턴에서 설정 파일이나 초기화 데이터를 메인 컨테이너에 전달하는 방법이다.

</details>

**Q2.** configMap 볼륨을 마운트한 상태에서 ConfigMap을 삭제하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

이미 마운트된 볼륨은 즉시 영향을 받지 않는다. kubelet이 로컬에 캐시한 내용으로 계속 서비스한다. 그러나 kubelet이 갱신을 시도할 때 ConfigMap이 없으면 에러가 발생하고, 파드가 재시작되거나 새 파드가 생성될 때 볼륨 마운트 단계에서 실패한다. 해당 ConfigMap을 사용하는 파드의 status에 `MountVolume.SetUp failed` 이벤트가 기록된다. 따라서 실행 중인 파드가 있는 한 ConfigMap을 삭제하면 안 된다.

</details>

**Q3.** 같은 파드의 두 컨테이너가 같은 secret 볼륨을 각각 마운트했을 때, 파일이 두 번 복사되는가?

<details>
<summary>해설 보기</summary>

아니다. secret 볼륨은 파드 단위로 하나만 생성되며, `/var/lib/kubelet/pods/<uid>/volumes/kubernetes.io~secret/<volume-name>/`에 한 번만 생성된다. 컨테이너들은 각자 자신의 namespace에서 해당 경로를 bind mount로 마운트한다. 실제 파일은 노드의 한 위치에만 존재하며, bind mount를 통해 여러 컨테이너 namespace에서 동일한 파일에 접근한다.

</details>

---

> ⬅️ 이전 챕터: [Ch3-07. Network Policy](../networking/07-network-policy.md)  
> ➡️ 다음: [02. PV와 PVC — 동적 프로비저닝](./02-pv-pvc-storageclass.md)
