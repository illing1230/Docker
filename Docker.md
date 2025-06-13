# Docker 명령어 치트시트

## 📋 목차
- [이미지 관리](#이미지-관리)
- [컨테이너 관리](#컨테이너-관리)
- [빌드 관련](#빌드-관련)
- [네트워크 관리](#네트워크-관리)
- [볼륨 관리](#볼륨-관리)
- [시스템 관리](#시스템-관리)
- [Docker Compose](#docker-compose)
- [실용적인 예시](#실용적인-예시)

---

## 🖼️ 이미지 관리

### 이미지 조회
```bash
# 모든 이미지 목록
docker images
docker image ls

# 특정 이미지만 조회
docker images nginx
docker images "myapp:*"

# 이미지 상세 정보
docker inspect nginx:latest

# 이미지 히스토리 (레이어 정보)
docker history nginx:latest
```

### 이미지 다운로드
```bash
# 이미지 다운로드
docker pull nginx
docker pull nginx:alpine
docker pull node:18

# 특정 플랫폼용 이미지 다운로드
docker pull --platform linux/amd64 nginx
```

### 이미지 삭제
```bash
# 특정 이미지 삭제
docker rmi nginx:latest
docker rmi abc123def456

# 강제 삭제
docker rmi -f nginx:latest

# 댕글링 이미지 삭제 (태그 없는 이미지)
docker image prune

# 사용하지 않는 모든 이미지 삭제
docker image prune -a

# 24시간 이전 이미지 삭제
docker image prune -a --filter "until=24h"
```

### 이미지 태그 및 백업
```bash
# 이미지 태그 추가
docker tag myapp:latest myapp:v1.0
docker tag myapp:latest registry.com/myapp:latest

# 이미지 백업
docker save nginx:latest > nginx-backup.tar
docker save -o nginx-backup.tar nginx:latest

# 백업에서 복원
docker load < nginx-backup.tar
docker load -i nginx-backup.tar
```

---

## 🚢 컨테이너 관리

### 컨테이너 실행
```bash
# 기본 실행
docker run nginx
docker run hello-world

# 백그라운드 실행
docker run -d nginx

# 인터랙티브 모드
docker run -it ubuntu bash
docker run -it --rm ubuntu bash  # 종료 시 자동 삭제

# 포트 바인딩
docker run -d -p 8080:80 nginx
docker run -d -p 127.0.0.1:8080:80 nginx

# 볼륨 마운트
docker run -d -v /host/path:/container/path nginx
docker run -d -v $(pwd):/app node:18

# 환경 변수 설정
docker run -d -e NODE_ENV=production myapp
docker run -d --env-file .env myapp

# 이름 지정 및 재시작 정책
docker run -d --name my-nginx --restart=always nginx
```

### 컨테이너 조회
```bash
# 실행 중인 컨테이너
docker ps

# 모든 컨테이너 (정지된 것 포함)
docker ps -a

# 컨테이너 상세 정보
docker inspect container-name

# 컨테이너 리소스 사용량
docker stats
docker stats container-name
```

### 컨테이너 제어
```bash
# 컨테이너 정지
docker stop container-name
docker stop $(docker ps -q)  # 모든 실행 중인 컨테이너 정지

# 컨테이너 시작
docker start container-name

# 컨테이너 재시작
docker restart container-name

# 컨테이너 일시정지/재개
docker pause container-name
docker unpause container-name

# 컨테이너 강제 종료
docker kill container-name
```

### 컨테이너 삭제
```bash
# 컨테이너 삭제
docker rm container-name

# 강제 삭제 (실행 중이어도)
docker rm -f container-name

# 정지된 모든 컨테이너 삭제
docker container prune

# 모든 컨테이너 삭제
docker rm -f $(docker ps -aq)
```

### 컨테이너 내부 작업
```bash
# 컨테이너 내부 접속
docker exec -it container-name bash
docker exec -it container-name sh

# 컨테이너에서 명령어 실행
docker exec container-name ls -la
docker exec -it container-name npm install

# 로그 확인
docker logs container-name
docker logs -f container-name  # 실시간 로그
docker logs --tail 100 container-name  # 마지막 100줄
```

### 파일 복사
```bash
# 호스트 → 컨테이너
docker cp /host/file.txt container-name:/path/

# 컨테이너 → 호스트
docker cp container-name:/path/file.txt /host/

# 디렉토리 복사
docker cp /host/folder container-name:/path/
```

---

## 🔨 빌드 관련

### 이미지 빌드
```bash
# 기본 빌드
docker build .
docker build -t myapp .
docker build -t myapp:v1.0 .

# 다른 Dockerfile 사용
docker build -f Dockerfile.prod -t myapp:prod .

# 빌드 인자 전달
docker build --build-arg NODE_ENV=production -t myapp .

# 캐시 사용 안함
docker build --no-cache -t myapp .

# 멀티스테이지 빌드 특정 타겟
docker build --target production -t myapp:prod .

# 빌드 로그 상세 출력
docker build --progress=plain -t myapp .
```

### 빌드 컨텍스트 최적화
```bash
# .dockerignore 파일 예시
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
coverage
.nyc_output
```

### BuildKit 사용
```bash
# BuildKit 활성화
export DOCKER_BUILDKIT=1
docker build .

# 또는 일회성
DOCKER_BUILDKIT=1 docker build .
```

---

## 🌐 네트워크 관리

### 네트워크 조회
```bash
# 네트워크 목록
docker network ls

# 네트워크 상세 정보
docker network inspect bridge
```

### 네트워크 생성/삭제
```bash
# 네트워크 생성
docker network create my-network
docker network create --driver bridge my-bridge

# 네트워크 삭제
docker network rm my-network

# 사용하지 않는 네트워크 삭제
docker network prune
```

### 네트워크 연결
```bash
# 특정 네트워크로 컨테이너 실행
docker run -d --network=my-network nginx

# 기존 컨테이너를 네트워크에 연결
docker network connect my-network container-name

# 네트워크에서 분리
docker network disconnect my-network container-name
```

---

## 💾 볼륨 관리

### 볼륨 조회
```bash
# 볼륨 목록
docker volume ls

# 볼륨 상세 정보
docker volume inspect my-volume
```

### 볼륨 생성/삭제
```bash
# 볼륨 생성
docker volume create my-volume

# 볼륨 삭제
docker volume rm my-volume

# 사용하지 않는 볼륨 삭제
docker volume prune
```

### 볼륨 사용
```bash
# Named 볼륨 사용
docker run -d -v my-volume:/data nginx

# 호스트 디렉토리 바인드
docker run -d -v /host/path:/container/path nginx

# 읽기 전용 마운트
docker run -d -v /host/path:/container/path:ro nginx
```

---

## ⚙️ 시스템 관리

### 시스템 정보
```bash
# Docker 시스템 정보
docker system info
docker version

# 디스크 사용량
docker system df
docker system df -v  # 상세 정보
```

### 시스템 정리
```bash
# 전체 시스템 정리
docker system prune

# 이미지까지 포함해서 정리
docker system prune -a

# 볼륨까지 포함해서 정리
docker system prune -a --volumes

# 확인 없이 정리
docker system prune -a -f
```

### 개별 리소스 정리
```bash
# 정지된 컨테이너 정리
docker container prune

# 사용하지 않는 이미지 정리
docker image prune
docker image prune -a

# 사용하지 않는 네트워크 정리
docker network prune

# 사용하지 않는 볼륨 정리
docker volume prune
```

---

## 🐳 Docker Compose

### 기본 명령어
```bash
# 서비스 시작
docker-compose up
docker-compose up -d  # 백그라운드

# 특정 서비스만 시작
docker-compose up web

# 빌드와 함께 시작
docker-compose up --build

# 서비스 정지
docker-compose down

# 볼륨까지 삭제
docker-compose down -v

# 이미지까지 삭제
docker-compose down --rmi all
```

### 서비스 관리
```bash
# 서비스 상태 확인
docker-compose ps

# 로그 확인
docker-compose logs
docker-compose logs -f web

# 서비스 재시작
docker-compose restart
docker-compose restart web

# 서비스 스케일링
docker-compose up -d --scale web=3
```

### 실행 중인 서비스 작업
```bash
# 서비스 내부 접속
docker-compose exec web bash

# 서비스에서 명령어 실행
docker-compose exec web npm install

# 일회성 컨테이너 실행
docker-compose run web npm test
```

---

## 💡 실용적인 예시

### 웹 애플리케이션 배포
```bash
# Node.js 앱 실행
docker run -d \
  --name my-app \
  -p 3000:3000 \
  -e NODE_ENV=production \
  -v $(pwd):/app \
  -w /app \
  node:18 npm start

# Nginx 웹서버
docker run -d \
  --name web-server \
  -p 80:80 \
  -v $(pwd)/html:/usr/share/nginx/html \
  nginx:alpine
```

### 데이터베이스 실행
```bash
# MySQL
docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=myapp \
  -p 3306:3306 \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0

# PostgreSQL
docker run -d \
  --name postgres-db \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15

# Redis
docker run -d \
  --name redis-cache \
  -p 6379:6379 \
  redis:alpine
```

### 개발 환경 설정
```bash
# 개발용 Node.js 환경
docker run -it --rm \
  -v $(pwd):/app \
  -w /app \
  -p 3000:3000 \
  node:18 bash

# Python 개발 환경
docker run -it --rm \
  -v $(pwd):/app \
  -w /app \
  -p 8000:8000 \
  python:3.11 bash
```

### 정리 스크립트
```bash
#!/bin/bash
# docker-cleanup.sh

echo "Docker 시스템 정리 시작..."

# 정지된 컨테이너 삭제
docker container prune -f

# 댕글링 이미지 삭제
docker image prune -f

# 사용하지 않는 네트워크 삭제
docker network prune -f

# 30일 이상된 미사용 이미지 삭제
docker image prune -a -f --filter "until=720h"

echo "정리 완료!"
docker system df
```

---

## 🔧 유용한 팁

### 별칭(Alias) 설정
```bash
# ~/.bashrc 또는 ~/.zshrc에 추가
alias dps='docker ps'
alias dpsa='docker ps -a'
alias di='docker images'
alias dex='docker exec -it'
alias dlog='docker logs -f'
alias dstop='docker stop $(docker ps -q)'
alias drm='docker rm $(docker ps -aq)'
alias drmi='docker rmi $(docker images -q)'
```

### Docker 명령어 자동완성
```bash
# Ubuntu/Debian
sudo apt-get install bash-completion

# macOS (Homebrew)
brew install bash-completion

# zsh 사용자
echo 'fpath=(~/.zsh/completion $fpath)' >> ~/.zshrc
echo 'autoload -Uz compinit && compinit' >> ~/.zshrc
```

### 자주 사용하는 조합
```bash
# 모든 컨테이너 정지 후 삭제
docker stop $(docker ps -q) && docker rm $(docker ps -aq)

# 모든 이미지 삭제
docker rmi $(docker images -q)

# 시스템 전체 정리 (주의!)
docker system prune -a --volumes -f

# 특정 패턴의 컨테이너만 정리
docker rm $(docker ps -a | grep "temp-" | awk '{print $1}')
```

---

## 📚 추가 리소스

- [Docker 공식 문서](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Dockerfile 베스트 프랙티스](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Compose 문서](https://docs.docker.com/compose/)

> **⚠️ 주의사항**
> - `docker system prune -a --volumes -f` 같은 명령어는 모든 데이터를 삭제할 수 있으니 주의하세요
> - 프로덕션 환경에서는 항상 백업을 먼저 수행하세요
> - 중요한 데이터는 볼륨을 사용하여 영속화하세요
