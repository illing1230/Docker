# Dockerfile 명령어 완벽 가이드

## 📋 목차
- [기본 명령어](#기본-명령어)
- [파일 및 디렉토리 작업](#파일-및-디렉토리-작업)
- [실행 관련 명령어](#실행-관련-명령어)
- [환경 설정](#환경-설정)
- [네트워크 및 볼륨](#네트워크-및-볼륨)
- [메타데이터](#메타데이터)
- [멀티스테이지 빌드](#멀티스테이지-빌드)
- [실제 사용 예시](#실제-사용-예시)
- [베스트 프랙티스](#베스트-프랙티스)

---

## 🏗️ 기본 명령어

### FROM - 베이스 이미지 지정
```dockerfile
# 기본 사용법
FROM ubuntu:20.04
FROM node:18
FROM python:3.11-alpine

# 특정 플랫폼 지정
FROM --platform=linux/amd64 node:18

# 멀티스테이지 빌드에서 별칭 사용
FROM node:18 AS builder
FROM nginx:alpine AS production

# 베이스 이미지에 태그 지정
FROM node:18.17.0
FROM ubuntu:focal-20230605
```

### WORKDIR - 작업 디렉토리 설정
```dockerfile
# 작업 디렉토리 설정
WORKDIR /app
WORKDIR /usr/src/app
WORKDIR /var/www/html

# 상대 경로 사용 (이전 WORKDIR 기준)
WORKDIR /app
WORKDIR backend  # /app/backend가 됨

# 변수 사용
ARG APP_HOME=/application
WORKDIR $APP_HOME
```

---

## 📁 파일 및 디렉토리 작업

### COPY - 파일/디렉토리 복사
```dockerfile
# 기본 사용법
COPY . .
COPY src/ /app/src/
COPY package.json /app/

# 여러 파일 복사
COPY package.json package-lock.json ./
COPY *.json ./

# 소유자 변경과 함께 복사
COPY --chown=node:node . .
COPY --chown=1000:1000 app/ /app/

# 특정 스테이지에서 복사 (멀티스테이지)
COPY --from=builder /app/dist /usr/share/nginx/html
COPY --from=node:18 /usr/local/bin/node /usr/local/bin/

# 권한 설정과 함께 복사
COPY --chmod=755 scripts/start.sh /usr/local/bin/
```

### ADD - 고급 파일 복사
```dockerfile
# URL에서 파일 다운로드
ADD https://github.com/user/repo/archive/main.tar.gz /tmp/

# tar 파일 자동 압축 해제
ADD app.tar.gz /app/

# 일반 파일 복사 (COPY와 동일)
ADD config.json /app/

# 권한 설정
ADD --chown=www-data:www-data app.tar.gz /var/www/
```

---

## ⚙️ 실행 관련 명령어

### RUN - 빌드 시 명령어 실행
```dockerfile
# 기본 사용법
RUN apt-get update
RUN npm install
RUN pip install -r requirements.txt

# 여러 명령어 연결 (레이어 최적화)
RUN apt-get update && \
    apt-get install -y curl wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 패키지 매니저별 예시
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    nodejs \
    npm \
    && rm -rf /var/lib/apt/lists/*

RUN yum update -y && \
    yum install -y git curl && \
    yum clean all

RUN apk add --no-cache \
    python3 \
    py3-pip \
    nodejs \
    npm

# 캐시 마운트 사용 (BuildKit)
RUN --mount=type=cache,target=/root/.npm \
    npm install

RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y python3
```

### CMD - 컨테이너 실행 시 기본 명령어
```dockerfile
# exec 형태 (권장)
CMD ["node", "app.js"]
CMD ["python", "main.py"]
CMD ["nginx", "-g", "daemon off;"]

# shell 형태
CMD node app.js
CMD python main.py

# ENTRYPOINT와 함께 사용 (기본 인자)
ENTRYPOINT ["python"]
CMD ["app.py"]
```

### ENTRYPOINT - 컨테이너 진입점
```dockerfile
# exec 형태 (권장)
ENTRYPOINT ["python", "app.py"]
ENTRYPOINT ["./start.sh"]
ENTRYPOINT ["java", "-jar", "app.jar"]

# shell 형태
ENTRYPOINT python app.py

# CMD와 조합
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run image script.py 실행 시 python script.py가 됨

# 스크립트 활용
COPY docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["python", "app.py"]
```

---

## 🌍 환경 설정

### ENV - 환경 변수 설정
```dockerfile
# 기본 사용법
ENV NODE_ENV=production
ENV PORT=3000
ENV DEBUG=false

# 여러 환경 변수 한번에
ENV NODE_ENV=production \
    PORT=3000 \
    DEBUG=false

# 변수 참조
ENV APP_HOME=/app
ENV PATH=$APP_HOME/bin:$PATH

# 런타임에 변경 가능
ENV DATABASE_URL=postgresql://localhost/myapp
```

### ARG - 빌드 시 인자
```dockerfile
# 기본 값이 있는 빌드 인자
ARG NODE_VERSION=18
ARG BUILD_ENV=development

# 기본 값이 없는 빌드 인자
ARG API_KEY
ARG BUILD_NUMBER

# FROM에서 사용
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}

# ENV와 조합
ARG BUILD_ENV=production
ENV NODE_ENV=$BUILD_ENV

# 멀티스테이지에서 전역 ARG
ARG NODEJS_VERSION=18

FROM node:${NODEJS_VERSION} AS builder
# ARG를 다시 선언해야 이 스테이지에서 사용 가능
ARG NODEJS_VERSION
RUN echo "Node version: ${NODEJS_VERSION}"
```

### USER - 실행 사용자 설정
```dockerfile
# 사용자 생성 후 전환
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001
USER nextjs

# 기존 사용자 사용
USER node
USER www-data
USER 1000

# root로 다시 전환
USER root
RUN apt-get update
USER node
```

---

## 🔌 네트워크 및 볼륨

### EXPOSE - 포트 노출
```dockerfile
# 단일 포트
EXPOSE 3000
EXPOSE 80
EXPOSE 443

# 여러 포트
EXPOSE 3000 3001 3002

# UDP 포트
EXPOSE 53/udp

# 변수 사용
ARG PORT=3000
EXPOSE $PORT
```

### VOLUME - 볼륨 마운트 포인트
```dockerfile
# 단일 볼륨
VOLUME /data
VOLUME /var/log

# 여러 볼륨
VOLUME ["/data", "/var/log", "/tmp"]

# 애플리케이션별 예시
VOLUME /var/lib/mysql        # MySQL 데이터
VOLUME /var/lib/postgresql   # PostgreSQL 데이터
VOLUME /usr/share/nginx/html # Nginx 웹 콘텐츠
```

---

## 📝 메타데이터

### LABEL - 메타데이터 라벨
```dockerfile
# 기본 사용법
LABEL version="1.0"
LABEL description="My Application"

# 여러 라벨 한번에
LABEL version="1.0" \
      description="My Application" \
      maintainer="user@example.com"

# 표준 라벨들
LABEL org.opencontainers.image.title="My App"
LABEL org.opencontainers.image.description="Application description"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.authors="John Doe <john@example.com>"
LABEL org.opencontainers.image.source="https://github.com/user/repo"
LABEL org.opencontainers.image.licenses="MIT"
```

### MAINTAINER - 유지보수자 (비권장, LABEL 사용 권장)
```dockerfile
# 비권장 (deprecated)
MAINTAINER John Doe <john@example.com>

# 권장 방법
LABEL maintainer="John Doe <john@example.com>"
```

---

## 🔧 멀티스테이지 빌드

### 기본 멀티스테이지
```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM nginx:alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 복잡한 멀티스테이지
```dockerfile
# Base stage
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./

# Dependencies stage
FROM base AS deps
RUN npm ci --only=production && npm cache clean --force

# Build stage
FROM base AS build
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package*.json ./
EXPOSE 3000
USER node
CMD ["npm", "start"]
```

---

## 💡 실제 사용 예시

### Node.js 애플리케이션
```dockerfile
# Node.js + npm
FROM node:18-alpine

# 보안을 위한 사용자 생성
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

WORKDIR /app

# 의존성 파일 먼저 복사 (캐시 최적화)
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# 소스 코드 복사
COPY --chown=nextjs:nodejs . .

USER nextjs

EXPOSE 3000

ENV NODE_ENV=production
ENV PORT=3000

CMD ["npm", "start"]
```

### Python 애플리케이션
```dockerfile
# Python + pip
FROM python:3.11-slim

# 시스템 패키지 업데이트 및 설치
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# 작업 디렉토리 설정
WORKDIR /app

# 의존성 파일 복사 및 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 코드 복사
COPY . .

# 비root 사용자 생성 및 전환
RUN useradd --create-home --shell /bin/bash app
USER app

EXPOSE 8000

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

CMD ["python", "app.py"]
```

### Java 애플리케이션
```dockerfile
# 멀티스테이지 Java 빌드
FROM maven:3.8-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

FROM openjdk:17-jre-slim
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

RUN useradd --create-home --shell /bin/bash app
USER app

EXPOSE 8080

ENV JAVA_OPTS="-Xmx512m -Xms256m"

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### React 애플리케이션 (Nginx)
```dockerfile
# 멀티스테이지 React 빌드
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Database (MySQL)
```dockerfile
FROM mysql:8.0

# 환경 변수 설정
ENV MYSQL_ROOT_PASSWORD=rootpassword
ENV MYSQL_DATABASE=myapp
ENV MYSQL_USER=appuser
ENV MYSQL_PASSWORD=apppassword

# 초기화 스크립트 복사
COPY init.sql /docker-entrypoint-initdb.d/

# 설정 파일 복사
COPY my.cnf /etc/mysql/conf.d/

EXPOSE 3306

VOLUME /var/lib/mysql
```

---

## 🎯 베스트 프랙티스

### 레이어 최적화
```dockerfile
# ❌ 나쁜 예 - 레이어가 많음
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y pip
RUN apt-get clean

# ✅ 좋은 예 - 하나의 레이어
RUN apt-get update && \
    apt-get install -y python3 pip && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 캐시 최적화
```dockerfile
# ✅ 의존성 파일을 먼저 복사하여 캐시 활용
COPY package*.json ./
RUN npm install

# 소스 코드는 나중에 복사
COPY . .
```

### 보안 고려사항
```dockerfile
# ✅ 비root 사용자 사용
RUN adduser --disabled-password --gecos '' appuser
USER appuser

# ✅ 최소한의 권한으로 파일 복사
COPY --chown=appuser:appuser . .

# ✅ 불필요한 패키지 설치 피하기
RUN apt-get update && \
    apt-get install -y --no-install-recommends python3 && \
    rm -rf /var/lib/apt/lists/*
```

### .dockerignore 활용
```dockerfile
# .dockerignore 파일 예시
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.DS_Store
coverage
.nyc_output
```

### 환경별 설정
```dockerfile
# 개발/프로덕션 구분
ARG BUILD_ENV=production
ENV NODE_ENV=$BUILD_ENV

# 조건부 패키지 설치
RUN if [ "$BUILD_ENV" = "development" ] ; then npm install ; \
    else npm ci --only=production ; fi
```

### 헬스체크 추가
```dockerfile
# 헬스체크 설정
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# HTTP 서버용
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/ || exit 1
```

---

## 🔍 디버깅 및 최적화

### 빌드 시점 디버깅
```dockerfile
# 중간 결과 확인
RUN ls -la /app
RUN echo "Current user: $(whoami)"
RUN echo "Environment: $NODE_ENV"

# 파일 내용 확인
RUN cat package.json
```

### 이미지 크기 최적화
```dockerfile
# ✅ Alpine 이미지 사용
FROM node:18-alpine

# ✅ 멀티스테이지 빌드로 크기 줄이기
FROM node:18 AS builder
# 빌드 과정...

FROM node:18-alpine
COPY --from=builder /app/dist ./

# ✅ 불필요한 파일 삭제
RUN npm install && npm cache clean --force
```

### 빌드 속도 최적화
```dockerfile
# ✅ BuildKit 캐시 마운트 사용
RUN --mount=type=cache,target=/root/.npm \
    npm install

# ✅ 병렬 RUN 명령어
RUN apt-get update && apt-get install -y \
    python3 \
    nodejs \
    && rm -rf /var/lib/apt/lists/*
```

---

## 📚 참고 자료

### 유용한 베이스 이미지들
```dockerfile
# 최소 크기
FROM alpine:latest
FROM scratch

# 언어별 공식 이미지
FROM node:18-alpine
FROM python:3.11-slim
FROM openjdk:17-jre-slim
FROM golang:1.19-alpine
FROM php:8.1-fpm-alpine

# 웹서버
FROM nginx:alpine
FROM httpd:alpine
FROM traefik:latest
```

### 자주 사용하는 패키지 설치
```dockerfile
# Ubuntu/Debian 계열
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    git \
    vim \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Alpine 계열
RUN apk add --no-cache \
    curl \
    wget \
    git \
    vim \
    build-base

# CentOS/RHEL 계열
RUN yum update -y && yum install -y \
    curl \
    wget \
    git \
    vim \
    gcc \
    && yum clean all
```

---

## ⚠️ 주의사항

1. **보안**: 항상 비root 사용자로 애플리케이션 실행
2. **크기**: 불필요한 패키지 설치 피하고 정리 명령어 추가
3. **캐시**: 자주 변경되는 파일은 나중에 COPY
4. **비밀정보**: ARG나 ENV에 민감한 정보 저장 금지
5. **레이어**: RUN 명령어를 연결하여 레이어 수 최소화

이 가이드를 참고하여 효율적이고 안전한 Dockerfile을 작성하세요!
