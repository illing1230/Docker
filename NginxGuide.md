Nginx에 대해 자세히 설명해드릴게요. 특히 Svelte + FastAPI 구조에서 왜 필요한지부터 시작해보겠습니다.Nginx에 대한 완전한 가이드를 만들어드렸습니다!

**핵심 요약:**

**왜 Nginx가 필요한가?**
1. **단일 진입점**: 프론트엔드와 백엔드를 하나의 도메인으로 통합
2. **CORS 문제 해결**: 같은 도메인에서 모든 요청 처리
3. **성능 최적화**: 정적 파일은 Nginx가 직접 서빙 (FastAPI보다 빠름)
4. **보안 강화**: 백엔드를 외부에서 직접 접근 불가
5. **SSL 통합 관리**: 하나의 지점에서 HTTPS 설정

**Svelte + FastAPI에서 특히 중요한 설정:**
- `try_files $uri $uri/ /index.html;` → SPA 라우팅 지원
- `/api/` 경로를 백엔드로 프록시
- 정적 파일 캐싱 최적화
- CORS 헤더 설정 (필요시)

**실제 적용할 때 주의사항:**
1. **포트 매핑**: Docker에서 서비스 간 통신 설정
2. **경로 설정**: `/api/` → `http://backend:8000/` (슬래시 주의)
3. **캐싱 전략**: 정적 파일은 장기 캐싱, HTML은 캐싱 안 함
4. **보안 헤더**: 프로덕션에서는 보안 헤더 필수

어떤 부분을 먼저 적용해보고 싶으신가요? 실제 설정하면서 막히는 부분이 있으면 언제든 질문해 주세요!


# Nginx 완전 가이드 - 웹 애플리케이션에서의 필요성과 설정

## 1. Nginx란 무엇인가?

Nginx(엔진엑스)는 고성능 웹 서버, 리버스 프록시, 로드 밸런서입니다. 러시아의 Igor Sysoev가 개발했으며, 높은 동시 접속 처리 능력과 낮은 메모리 사용량으로 유명합니다.

### 주요 특징
- **비동기 이벤트 기반 아키텍처**: 수만 개의 동시 연결을 효율적으로 처리
- **낮은 메모리 사용량**: Apache 대비 현저히 적은 리소스 사용
- **높은 성능**: 정적 파일 서빙에서 뛰어난 성능
- **유연한 설정**: 다양한 용도로 활용 가능

## 2. 왜 Nginx가 필요한가?

### 2.1 Svelte + FastAPI 구조에서의 문제점

**Nginx 없이 직접 서빙할 때의 한계:**

```
브라우저 → Svelte Dev Server (포트 5173)
브라우저 → FastAPI Server (포트 8000)
```

**문제점들:**
1. **CORS 이슈**: 다른 포트 간 통신 시 발생
2. **프로덕션 부적합**: 개발 서버는 프로덕션용이 아님
3. **성능 문제**: 정적 파일 서빙에 최적화되지 않음
4. **보안 취약점**: 직접 노출 시 보안 위험
5. **SSL/HTTPS 설정 복잡**: 각 서버마다 개별 설정 필요

### 2.2 Nginx 도입 후의 개선점

```
브라우저 → Nginx (포트 80/443) → {
    정적 파일 → 직접 서빙
    /api → FastAPI Server (포트 8000)
}
```

**장점들:**
- **단일 진입점**: 하나의 도메인으로 모든 요청 처리
- **CORS 해결**: 같은 도메인에서 모든 요청 처리
- **성능 향상**: 정적 파일은 Nginx가 직접 서빙
- **보안 강화**: 백엔드 서버를 외부에서 직접 접근 불가
- **SSL 통합 관리**: 하나의 지점에서 HTTPS 설정

## 3. Nginx의 주요 역할들

### 3.1 웹 서버 (Static File Server)
```nginx
server {
    listen 80;
    
    # 정적 파일 서빙
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;  # SPA 라우팅 지원
    }
    
    # 캐시 설정
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### 3.2 리버스 프록시 (Reverse Proxy)
```nginx
server {
    listen 80;
    
    # API 요청을 백엔드로 전달
    location /api/ {
        proxy_pass http://backend:8000/;
        
        # 프록시 헤더 설정
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 타임아웃 설정
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

### 3.3 로드 밸런서 (Load Balancer)
```nginx
# 업스트림 서버 정의
upstream backend_servers {
    server backend1:8000 weight=3;
    server backend2:8000 weight=2;
    server backend3:8000 weight=1;
    
    # 헬스 체크 (Nginx Plus 기능)
    # health_check;
}

server {
    listen 80;
    
    location /api/ {
        proxy_pass http://backend_servers/;
    }
}
```

## 4. 상세 설정 가이드

### 4.1 기본 구조 이해
```nginx
# 전역 설정
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;

# 이벤트 블록
events {
    worker_connections 1024;
    use epoll;  # Linux에서 성능 향상
}

# HTTP 블록
http {
    # MIME 타입 설정
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 로그 형식
    log_format main '$remote_addr - $remote_user [$time_local] '
                   '"$request" $status $body_bytes_sent '
                   '"$http_referer" "$http_user_agent"';
    
    # 서버 블록들
    server {
        # 서버 설정
    }
}
```

### 4.2 Svelte + FastAPI용 완전한 설정

```nginx
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 기본 설정
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # Gzip 압축
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/javascript
        application/xml+rss
        application/json;
    
    # 업스트림 서버 정의
    upstream fastapi_backend {
        server backend:8000;
        # 여러 서버가 있을 경우
        # server backend2:8000;
        # server backend3:8000;
    }
    
    server {
        listen 80;
        server_name localhost;
        
        # 보안 헤더
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        
        # 정적 파일 서빙 (Svelte 빌드 결과물)
        location / {
            root /usr/share/nginx/html;
            index index.html;
            
            # SPA 라우팅 지원 (중요!)
            try_files $uri $uri/ /index.html;
            
            # 캐시 설정
            location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
                access_log off;
            }
            
            # HTML 파일은 캐시하지 않음
            location ~* \.html$ {
                expires -1;
                add_header Cache-Control "no-cache, no-store, must-revalidate";
            }
        }
        
        # API 요청을 FastAPI로 프록시
        location /api/ {
            # 슬래시 제거하여 전달 (/api/users → /users)
            proxy_pass http://fastapi_backend/;
            
            # 프록시 헤더 설정
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # WebSocket 지원 (필요한 경우)
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            
            # CORS 헤더 (필요한 경우)
            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
            add_header Access-Control-Allow-Headers "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization";
            
            # Preflight 요청 처리
            if ($request_method = 'OPTIONS') {
                add_header Access-Control-Allow-Origin *;
                add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
                add_header Access-Control-Allow-Headers "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization";
                add_header Access-Control-Max-Age 1728000;
                add_header Content-Type 'text/plain; charset=utf-8';
                add_header Content-Length 0;
                return 204;
            }
            
            # 타임아웃 설정
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
            
            # 버퍼 설정
            proxy_buffering on;
            proxy_buffer_size 128k;
            proxy_buffers 4 256k;
            proxy_busy_buffers_size 256k;
        }
        
        # 헬스체크 엔드포인트
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # 에러 페이지 커스터마이징
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }
}
```

### 4.3 HTTPS/SSL 설정

```nginx
server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    # SSL 인증서 설정
    ssl_certificate /etc/ssl/certs/your-cert.pem;
    ssl_certificate_key /etc/ssl/private/your-key.pem;
    
    # SSL 설정 강화
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # HSTS (HTTP Strict Transport Security)
    add_header Strict-Transport-Security "max-age=63072000" always;
    
    # 나머지 설정은 위와 동일
    # ...
}

# HTTP to HTTPS 리다이렉트
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}
```

## 5. Docker에서의 Nginx 설정

### 5.1 Dockerfile
```dockerfile
FROM nginx:alpine

# 커스텀 설정 파일 복사
COPY nginx.conf /etc/nginx/nginx.conf

# Svelte 빌드 결과물 복사
COPY --from=builder /app/dist /usr/share/nginx/html

# 포트 노출
EXPOSE 80

# Nginx 실행
CMD ["nginx", "-g", "daemon off;"]
```

### 5.2 개발 환경용 설정
```nginx
# 개발 환경에서는 프록시 패스를 localhost로
location /api/ {
    proxy_pass http://host.docker.internal:8000/;
    
    # 또는 Docker Compose 서비스명 사용
    # proxy_pass http://backend:8000/;
}
```

## 6. 성능 최적화 팁

### 6.1 캐싱 전략
```nginx
# 정적 자산 장기 캐싱
location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    access_log off;
}

# API 응답 캐싱 (주의해서 사용)
location /api/static-data/ {
    proxy_pass http://fastapi_backend/static-data/;
    proxy_cache my_cache;
    proxy_cache_valid 200 5m;
    proxy_cache_use_stale error timeout updating;
}
```

### 6.2 압축 설정
```nginx
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_comp_level 6;
gzip_types
    text/plain
    text/css
    text/xml
    text/javascript
    application/javascript
    application/xml+rss
    application/json
    application/wasm;
```

### 6.3 연결 최적화
```nginx
# Keep-alive 연결
keepalive_timeout 65;
keepalive_requests 100;

# TCP 최적화
tcp_nopush on;
tcp_nodelay on;

# 파일 전송 최적화
sendfile on;
```

## 7. 보안 설정

### 7.1 기본 보안 헤더
```nginx
# XSS 보호
add_header X-XSS-Protection "1; mode=block" always;

# 콘텐츠 타입 스니핑 방지
add_header X-Content-Type-Options "nosniff" always;

# 클릭재킹 방지
add_header X-Frame-Options "SAMEORIGIN" always;

# CSP (Content Security Policy)
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;

# 추천인 정책
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

### 7.2 접근 제한
```nginx
# IP 기반 접근 제한
location /admin/ {
    allow 192.168.1.0/24;
    deny all;
    
    proxy_pass http://fastapi_backend/admin/;
}

# Rate Limiting
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location /api/ {
    limit_req zone=api burst=20 nodelay;
    proxy_pass http://fastapi_backend/;
}
```

## 8. 로그 및 모니터링

### 8.1 로그 설정
```nginx
# 커스텀 로그 형식
log_format detailed '$remote_addr - $remote_user [$time_local] '
                   '"$request" $status $body_bytes_sent '
                   '"$http_referer" "$http_user_agent" '
                   '$request_time $upstream_response_time';

# 접근 로그
access_log /var/log/nginx/access.log detailed;

# 에러 로그
error_log /var/log/nginx/error.log warn;
```

### 8.2 상태 모니터링
```nginx
# Nginx 상태 페이지 (stub_status 모듈 필요)
location /nginx_status {
    stub_status on;
    access_log off;
    allow 127.0.0.1;
    deny all;
}
```

## 9. 문제 해결 가이드

### 9.1 일반적인 문제들

**502 Bad Gateway**
- 백엔드 서버가 실행되지 않음
- 프록시 설정 오류
- 방화벽 차단

**404 Not Found (SPA 라우팅)**
```nginx
# 해결: try_files 설정 추가
try_files $uri $uri/ /index.html;
```

**CORS 오류**
```nginx
# 해결: CORS 헤더 추가
add_header Access-Control-Allow-Origin *;
```

### 9.2 디버깅 명령어
```bash
# 설정 파일 문법 검사
nginx -t

# 설정 다시 로드
nginx -s reload

# 로그 실시간 확인
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# 프로세스 확인
ps aux | grep nginx
```

## 10. 최적화된 프로덕션 설정 예시

```nginx
# /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # 기본 최적화
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    
    # 타임아웃
    keepalive_timeout 30;
    client_body_timeout 10;
    client_header_timeout 10;
    send_timeout 10;
    
    # 버퍼 크기
    client_body_buffer_size 128k;
    client_header_buffer_size 1k;
    client_max_body_size 10m;
    large_client_header_buffers 2 1k;
    
    # Gzip
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml text/javascript;
    
    # 로그
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for"';
    
    # 서버 설정 include
    include /etc/nginx/conf.d/*.conf;
}
```

이제 Nginx의 필요성과 설정에 대해 포괄적으로 이해하셨을 것입니다. 특정 부분에 대해 더 자세히 알고 싶으시거나 실제 구현 중 문제가 발생하면 언제든 말씀해 주세요!
