# Svelte + FastAPI Docker & Kubernetes 배포 가이드

## 1. 프로젝트 구조 예시
```
my-web-app/
├── frontend/          # Svelte 프로젝트
│   ├── src/
│   ├── package.json
│   ├── vite.config.js
│   └── Dockerfile
├── backend/           # FastAPI 프로젝트
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── docker-compose.yml
└── k8s/
    ├── frontend-deployment.yaml
    ├── backend-deployment.yaml
    └── ingress.yaml
```

## 2. Docker 설정

### Frontend Dockerfile (frontend/Dockerfile)
```dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Frontend Nginx 설정 (frontend/nginx.conf)
```nginx
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
        
        # API 요청을 백엔드로 프록시
        location /api/ {
            proxy_pass http://backend:8000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

### Backend Dockerfile (backend/Dockerfile)
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 시스템 패키지 업데이트 및 필요한 패키지 설치
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Python 의존성 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# 포트 노출
EXPOSE 8000

# 애플리케이션 실행
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Backend requirements.txt 예시
```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
python-multipart==0.0.6
# 필요한 다른 패키지들 추가
```

## 3. Docker Compose 설정

### docker-compose.yml
```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    container_name: my-app-backend
    ports:
      - "8000:8000"
    environment:
      - ENVIRONMENT=development
    volumes:
      - ./backend:/app
    networks:
      - app-network

  frontend:
    build: ./frontend
    container_name: my-app-frontend
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

## 4. 로컬 Docker 실행

### 이미지 빌드 및 실행
```bash
# 모든 서비스 빌드 및 실행
docker-compose up --build

# 백그라운드에서 실행
docker-compose up -d --build

# 로그 확인
docker-compose logs -f

# 서비스 중지
docker-compose down
```

### 개별 서비스 빌드
```bash
# 백엔드만 빌드
docker build -t my-app-backend ./backend

# 프론트엔드만 빌드
docker build -t my-app-frontend ./frontend

# 개별 실행
docker run -p 8000:8000 my-app-backend
docker run -p 3000:80 my-app-frontend
```

## 5. 이미지 레지스트리 푸시 (배포 준비)

### Docker Hub 사용 예시
```bash
# 태그 설정
docker tag my-app-backend your-dockerhub-username/my-app-backend:latest
docker tag my-app-frontend your-dockerhub-username/my-app-frontend:latest

# 푸시
docker push your-dockerhub-username/my-app-backend:latest
docker push your-dockerhub-username/my-app-frontend:latest
```

## 6. Kubernetes 배포

### Backend Deployment (k8s/backend-deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: your-dockerhub-username/my-app-backend:latest
        ports:
        - containerPort: 8000
        env:
        - name: ENVIRONMENT
          value: "production"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000
  type: ClusterIP
```

### Frontend Deployment (k8s/frontend-deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: your-dockerhub-username/my-app-frontend:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

### Ingress 설정 (k8s/ingress.yaml)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: your-domain.com  # 실제 도메인으로 변경
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

## 7. Kubernetes 배포 명령어

### 배포 실행
```bash
# 네임스페이스 생성 (선택사항)
kubectl create namespace my-app

# 모든 리소스 배포
kubectl apply -f k8s/ -n my-app

# 또는 개별 배포
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/frontend-deployment.yaml
kubectl apply -f k8s/ingress.yaml
```

### 배포 확인
```bash
# Pod 상태 확인
kubectl get pods -n my-app

# 서비스 확인
kubectl get services -n my-app

# Ingress 확인
kubectl get ingress -n my-app

# 로그 확인
kubectl logs -f deployment/backend-deployment -n my-app
kubectl logs -f deployment/frontend-deployment -n my-app
```

### 유용한 관리 명령어
```bash
# 배포 업데이트 (새 이미지 푸시 후)
kubectl rollout restart deployment/backend-deployment -n my-app
kubectl rollout restart deployment/frontend-deployment -n my-app

# 스케일링
kubectl scale deployment backend-deployment --replicas=3 -n my-app

# 포트 포워딩 (로컬 테스트용)
kubectl port-forward service/frontend-service 3000:80 -n my-app
kubectl port-forward service/backend-service 8000:8000 -n my-app

# 리소스 삭제
kubectl delete -f k8s/ -n my-app
```

## 8. 환경별 설정 팁

### 개발 환경
- `docker-compose`를 사용하여 빠른 개발
- 볼륨 마운트로 코드 변경사항 실시간 반영

### 프로덕션 환경
- 환경변수를 통한 설정 관리
- Secret을 사용한 민감한 정보 관리
- 리소스 제한 및 헬스체크 설정
- SSL/TLS 설정

### 모니터링 및 로깅
```bash
# 리소스 사용량 확인
kubectl top pods -n my-app
kubectl top nodes

# 이벤트 확인
kubectl get events -n my-app --sort-by=.metadata.creationTimestamp
```

이 가이드를 따라하시면 Svelte + FastAPI 애플리케이션을 Docker로 컨테이너화하고 Kubernetes에 배포할 수 있습니다.
