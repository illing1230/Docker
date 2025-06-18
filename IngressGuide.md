Kubernetes Ingress는 클러스터 외부에서 내부 서비스로 HTTP/HTTPS 트래픽을 라우팅하는 API 객체입니다. 쉽게 말해서 클러스터의 "관문(게이트웨이)" 역할을 한다고 생각하면 됩니다.

## Ingress가 필요한 이유

일반적으로 Kubernetes 서비스는 클러스터 내부에서만 접근 가능합니다. 외부에서 접근하려면 NodePort나 LoadBalancer 타입을 사용해야 하는데, 여러 서비스가 있을 때마다 각각 포트를 열거나 로드밸런서를 만드는 것은 비효율적입니다.

## Ingress의 주요 기능

**URL 기반 라우팅**: 같은 IP 주소로 들어온 요청을 URL 경로에 따라 다른 서비스로 분배합니다. 예를 들어:
- `example.com/api` → API 서비스
- `example.com/web` → 웹 서비스

**도메인 기반 라우팅**: 호스트명에 따라 다른 서비스로 요청을 보냅니다:
- `api.example.com` → API 서비스
- `web.example.com` → 웹 서비스

**SSL/TLS 종료**: HTTPS 인증서를 관리하고 SSL 암호화를 처리합니다.

**로드 밸런싱**: 여러 파드 간에 트래픽을 분산시킵니다.

## Ingress 구성 요소

Ingress는 두 부분으로 구성됩니다:

1. **Ingress 리소스**: 라우팅 규칙을 정의하는 YAML 설정
2. **Ingress Controller**: 실제로 트래픽을 처리하는 구현체 (nginx, traefik, istio 등)

## 간단한 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

이 설정은 `myapp.example.com`으로 들어오는 모든 요청을 `web-service`의 80번 포트로 전달합니다.

Ingress를 사용하면 하나의 진입점으로 여러 서비스를 효율적으로 관리할 수 있어서, 마이크로서비스 아키텍처에서 특히 유용합니다. 외부 사용자는 복잡한 내부 구조를 알 필요 없이 간단한 URL로 원하는 서비스에 접근할 수 있게 됩니다.
