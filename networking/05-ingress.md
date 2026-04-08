# Ingress — L7 라우팅과 Nginx Controller

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Ingress 리소스와 Ingress Controller는 어떻게 다르고, 둘의 관계는 무엇인가?
- Nginx Ingress Controller는 Ingress 오브젝트를 어떻게 `nginx.conf`로 변환하는가?
- 호스트 기반 라우팅과 경로 기반 라우팅은 내부적으로 어떻게 표현되는가?
- TLS 종료(Termination)는 어디서 처리되고, 백엔드까지 암호화하려면 어떻게 하는가?
- `kubectl logs`로 Ingress Controller의 요청 라우팅을 어떻게 추적하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Service를 LoadBalancer 타입으로 만들면 Service마다 클라우드 LB가 하나씩 생성된다. 10개 마이크로서비스라면 LB 10개의 비용이 발생한다. Ingress는 단일 LB 뒤에 도메인/경로 기반으로 여러 Service를 라우팅해 비용을 대폭 줄인다.

또한 TLS 인증서 관리, 요청 리다이렉션, Rate Limiting, 인증 연동 같은 횡단 관심사를 Ingress Controller 레벨에서 처리할 수 있어 애플리케이션 코드 수정이 불필요하다.

---

## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)

```
상황: Ingress를 생성했는데 도메인으로 접근이 안 됨

원리를 모를 때의 판단:
  "Ingress 설정이 잘못됐나?" → yaml 재확인 → 문제 없어 보임
  "DNS가 문제인가?" → DNS 확인 → 정상 설정
  → 원인 모름

실제 원인 파악 순서:
  1. Ingress Controller가 설치됐는가?
     kubectl get pods -n ingress-nginx
     → 없으면: helm install nginx-ingress 필요
     (Ingress 오브젝트만 만든다고 동작하지 않음)

  2. Ingress Controller의 External IP 확인
     kubectl get svc -n ingress-nginx
     → EXTERNAL-IP가 pending? → LoadBalancer 문제

  3. Ingress Controller 로그로 라우팅 확인
     kubectl logs -n ingress-nginx <controller-pod> | grep <host>

  4. Backend Service가 정상인가?
     kubectl get endpoints <service-name>
     → 비어있으면 selector 불일치
```

---

## ✨ 올바른 접근 (After — Ingress 아키텍처를 이해한 운영)

```
Ingress 구성 요소 이해:

  외부 클라이언트
       │
       ▼ DNS → Ingress Controller의 External IP
  LoadBalancer (클라우드 LB, 1개만 필요)
       │
       ▼ TCP:80/443
  Ingress Controller Pod (Nginx)
       │ Ingress 오브젝트를 읽어 nginx.conf 자동 생성
       ├─► app-a Service (host: a.example.com)
       ├─► app-b Service (host: b.example.com)
       └─► app-c Service (path: /api)

  Ingress 오브젝트 = 라우팅 규칙 선언 (YAML)
  Ingress Controller = 규칙을 실제 Nginx 설정으로 구현
```

---

## 🔬 내부 동작 원리

### Ingress Controller 설치와 역할

Ingress Controller는 별도로 설치해야 한다. 쿠버네티스 기본 설치에는 포함되지 않는다.

```bash
# Nginx Ingress Controller 설치 (Kind용)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# 설치 확인
kubectl get pods -n ingress-nginx
# ingress-nginx-controller-xxx   1/1   Running   (이 파드가 nginx 프로세스)

kubectl get svc -n ingress-nginx
# ingress-nginx-controller   LoadBalancer   10.96.x.x   <external-ip>   80:3xxxx/443:3xxxx/TCP
```

Ingress Controller는 쿠버네티스 API Server에 Ingress 오브젝트를 Watch한다. 새 Ingress가 생성되거나 변경되면 nginx.conf를 재생성하고 Nginx에 reload 신호를 보낸다.

### Ingress 오브젝트 → nginx.conf 변환

```yaml
# Ingress 정의
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-secret
  rules:
  - host: api.example.com          # 호스트 기반 라우팅
    http:
      paths:
      - path: /users               # 경로 기반 라우팅
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
  - host: admin.example.com        # 두 번째 호스트
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

이 Ingress를 Nginx Controller가 변환한 nginx.conf:

```nginx
# Ingress Controller가 자동 생성하는 nginx.conf (핵심 부분)

upstream user-service-80 {
    server 10.244.1.5:80;   # user-service 파드 IP
    server 10.244.1.6:80;
}

upstream order-service-80 {
    server 10.244.2.7:80;
}

server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate     /etc/nginx/ssl/tls.crt;  # TLS 종료 여기서 처리
    ssl_certificate_key /etc/nginx/ssl/tls.key;

    location /users {
        proxy_pass http://user-service-80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /orders {
        proxy_pass http://order-service-80;
    }
}

server {
    listen 443 ssl;
    server_name admin.example.com;
    # ...
}
```

Ingress Controller는 백엔드 Service의 Endpoints를 직접 Watch해 파드 IP 변경을 즉시 nginx upstream에 반영한다.

### TLS 종료(Termination)

```
TLS 종료 방식:

  방식 1: Ingress에서 종료 (가장 일반적)
    클라이언트 ─HTTPS─► Ingress Controller ─HTTP─► 백엔드 Service/Pod
    → Ingress Controller가 TLS 인증서 보유 및 복호화
    → 백엔드는 일반 HTTP 처리
    → 클러스터 내부 통신은 암호화 없음 (신뢰 경계 내부로 간주)

  방식 2: 파드까지 TLS 유지 (mTLS)
    클라이언트 ─HTTPS─► Ingress Controller ─HTTPS─► 백엔드 Pod
    → nginx annotation: nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    → 백엔드 파드도 TLS 설정 필요

TLS Secret 생성:
  kubectl create secret tls api-tls-secret \
    --cert=tls.crt \
    --key=tls.key

cert-manager와 연동 (자동 인증서 발급/갱신):
  - Let's Encrypt 인증서를 CertificateRequest → Certificate 오브젝트로 자동 관리
  - Ingress에 cert-manager annotation 추가 시 Secret 자동 생성
```

### Ingress Controller가 Endpoints를 직접 사용하는 이유

```
일반적인 예상:
  클라이언트 → Nginx → Service(ClusterIP) → kube-proxy(iptables) → 파드

실제 Nginx Ingress 동작:
  클라이언트 → Nginx → 파드 IP 직접 (kube-proxy 우회)

이유:
  kube-proxy의 iptables 로드밸런싱은 무작위 분배 + SNAT 발생
  Nginx가 직접 Endpoints를 Watch → 파드 IP 목록 보유
  Nginx 자체 upstream 로드밸런싱 사용 (세션 유지, 가중치 등 고급 기능)
  → kube-proxy iptables보다 Nginx의 L7 로드밸런싱이 더 정밀

확인 방법:
  kubectl logs -n ingress-nginx <controller-pod> | grep "upstream"
```

### 주요 Annotations

Nginx Ingress Controller는 Annotations로 세밀한 동작 제어를 지원한다.

```yaml
metadata:
  annotations:
    # 경로 재작성 (/api/users → /users)
    nginx.ingress.kubernetes.io/rewrite-target: /$1

    # Rate Limiting (분당 100 요청)
    nginx.ingress.kubernetes.io/limit-rps: "100"

    # 요청 크기 제한 (파일 업로드 등)
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # CORS 설정
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://frontend.example.com"

    # Basic Auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret

    # HTTP → HTTPS 강제 리다이렉트
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Connection Timeout
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
```

---

## 💻 실전 실험

### 1. Kind 클러스터에 Nginx Ingress 설치

```bash
# Kind 설정 (포트 매핑 필요)
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
  - containerPort: 443
    hostPort: 443
EOF

# Nginx Ingress Controller 설치
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# 설치 완료 대기
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### 2. 호스트 기반 라우팅 실습

```bash
# 두 개의 백엔드 서비스 생성
kubectl create deployment app-a --image=nginx
kubectl create deployment app-b --image=hashicorp/http-echo -- -text="App B"
kubectl expose deployment app-a --port=80
kubectl expose deployment app-b --port=5678

# Ingress 규칙 생성
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app-a.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-a
            port:
              number: 80
  - host: app-b.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-b
            port:
              number: 5678
EOF

# /etc/hosts에 임시 추가
echo "127.0.0.1 app-a.local app-b.local" | sudo tee -a /etc/hosts

# 라우팅 테스트
curl http://app-a.local      # nginx 응답
curl http://app-b.local      # "App B" 응답
```

### 3. 경로 기반 라우팅 실습

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: app-b
            port:
              number: 5678
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-a
            port:
              number: 80
EOF

curl http://myapp.local/       # app-a
curl http://myapp.local/api    # app-b
```

### 4. Nginx Controller 로그로 요청 라우팅 추적

```bash
# Controller 파드 이름 확인
CONTROLLER=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller \
  -o jsonpath='{.items[0].metadata.name}')

# 실시간 액세스 로그 (어느 Service로 라우팅됐는지 확인)
kubectl logs -n ingress-nginx $CONTROLLER -f | grep -v "health"

# 자동 생성된 nginx.conf 확인 (실제 변환 결과)
kubectl exec -n ingress-nginx $CONTROLLER -- cat /etc/nginx/nginx.conf | \
  grep -A5 "upstream\|server_name\|proxy_pass"
```

### 5. TLS 설정 실습

```bash
# 자체 서명 인증서 생성
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=secure.local/O=test"

# TLS Secret 생성
kubectl create secret tls secure-tls-secret --cert=tls.crt --key=tls.key

# TLS Ingress 생성
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.local
    secretName: secure-tls-secret
  rules:
  - host: secure.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-a
            port:
              number: 80
EOF

echo "127.0.0.1 secure.local" | sudo tee -a /etc/hosts
curl -k https://secure.local    # TLS 종료 확인
```

---

## 📊 Ingress Controller 비교

| 항목 | Nginx Ingress | Traefik | AWS ALB Controller |
|-----|-------------|--------|------------------|
| 설정 방식 | Annotation + ConfigMap | CRD(IngressRoute) | Annotation |
| TLS 자동 갱신 | cert-manager 연동 | Let's Encrypt 내장 | ACM 연동 |
| WebSocket | ✅ | ✅ | ✅ |
| gRPC | ✅ | ✅ | ✅ |
| Rate Limiting | ✅ Annotation | ✅ | ✅ |
| 클라우드 통합 | 범용 | 범용 | AWS 전용 |
| 성능 | 높음 | 높음 | 높음 |
| 러닝 커브 | 낮음 | 낮음 | AWS 지식 필요 |

---

## ⚖️ 트레이드오프

**Ingress vs Service LoadBalancer**

Ingress는 단일 LB로 여러 Service를 라우팅해 비용을 절약한다. 단, HTTP/HTTPS 기반(L7)이므로 TCP/UDP 직접 노출이 필요한 경우(DB, gRPC without HTTP, MQTT 등)에는 Service LoadBalancer를 써야 한다. 또한 Ingress Controller 자체가 단일 장애 지점이 될 수 있으므로 replicas를 2 이상으로 설정해야 한다.

**nginx.conf 재로드의 영향**

Ingress 규칙이 변경될 때마다 nginx.conf가 재생성되고 Nginx가 reload된다. Nginx reload는 graceful하지만 일시적으로 새 연결에 약간의 지연이 생긴다. Ingress 변경이 매우 빈번한 환경에서는 Lua 기반 동적 업스트림 설정을 지원하는 OpenResty Ingress를 고려할 수 있다.

---

## 📌 핵심 정리

```
Ingress = 라우팅 규칙 선언 (YAML 오브젝트)
Ingress Controller = 규칙을 실행하는 프로세스 (Nginx 파드)
→ 둘 다 있어야 동작 (Ingress만 만든다고 안 됨)

nginx.conf 자동 생성 흐름:
  Ingress 오브젝트 생성/변경
  → Controller가 Watch로 감지
  → nginx.conf 재생성 (server, upstream 블록)
  → Nginx graceful reload

라우팅 유형:
  호스트 기반: server_name api.example.com
  경로 기반:   location /users → upstream user-service

TLS 종료:
  Ingress Controller에서 처리 (Secret에서 인증서 로드)
  백엔드는 일반 HTTP로 통신 (클러스터 내부 신뢰)

비용 최적화:
  Service LoadBalancer 10개 → Ingress + LB 1개
  HTTP/HTTPS 서비스는 Ingress로 통합
  TCP/UDP 서비스는 별도 LoadBalancer 사용
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 호스트(`api.example.com`)에 두 개의 Ingress 오브젝트가 각각 다른 경로(`/users`, `/orders`)를 정의하고 있다. 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

Nginx Ingress Controller는 같은 호스트의 여러 Ingress 오브젝트를 병합해 하나의 `server` 블록으로 처리한다. `/users`와 `/orders` 경로 각각의 `location` 블록이 동일한 `server_name api.example.com` 블록에 들어간다. 단, 경로 충돌이 발생하면 먼저 생성된 Ingress의 규칙이 우선된다. 이 병합 동작은 Ingress Controller 구현마다 다를 수 있으므로, 하나의 호스트에 대한 규칙은 가능한 한 단일 Ingress 오브젝트에 정의하는 것이 권장된다.

</details>

**Q2.** Ingress Controller가 재시작될 때 기존 연결은 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Nginx는 `nginx -s reload`를 받으면 기존 worker 프로세스를 graceful하게 종료한다. 기존 연결은 처리를 완료한 후 종료되고, 새 worker 프로세스가 새 연결을 받는다. 단, Ingress Controller Pod 자체가 재시작(크래시 등)되면 Pod 재시작 시간 동안 새 연결이 처리되지 않는다. 이를 방지하려면 Ingress Controller를 `replicas: 2` 이상으로 설정하고, PodAntiAffinity로 다른 노드에 분산시켜야 한다.

</details>

**Q3.** cert-manager 없이 Let's Encrypt 인증서를 Ingress에 적용하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

수동으로 처리해야 한다. (1) Certbot으로 Let's Encrypt 인증서 발급. (2) `kubectl create secret tls my-cert --cert=fullchain.pem --key=privkey.pem`으로 Secret 생성. (3) Ingress의 `tls.secretName`에 지정. (4) 90일마다 수동 갱신 및 Secret 업데이트 필요. 이런 번거로움 때문에 실무에서는 cert-manager를 사용해 `Certificate` CRD로 자동 발급/갱신을 관리한다. cert-manager는 Ingress에 `cert-manager.io/cluster-issuer` annotation만 추가하면 자동으로 Secret을 생성하고 갱신한다.

</details>

---

<div align="center">

**[⬅️ 이전: Service 종류 완전 분해](./04-service-types.md)** | **[홈으로 🏠](../README.md)** | **[다음: CoreDNS — Service 이름 해석 ➡️](./06-coredns.md)**

</div>
