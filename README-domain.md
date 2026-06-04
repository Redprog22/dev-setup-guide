# 로컬 도메인 설정 가이드

WSL + Docker 개발 환경에서 포트 번호 없이 도메인과 서브도메인을 사용하는 방법을 안내합니다.

> 이 가이드는 [WSL + Docker 개발 환경 세팅 가이드](./README.md) 가 완료된 상태를 전제로 합니다.

---

## 목차

1. [이 가이드에서 만들 것](#1-이-가이드에서-만들-것)
2. [전체 구조 이해하기](#2-전체-구조-이해하기)
3. [Windows hosts 파일 수정](#3-windows-hosts-파일-수정)
4. [docker-compose.yml 수정](#4-docker-composeyml-수정)
5. [Nginx 리버스 프록시 설정](#5-nginx-리버스-프록시-설정)
6. [실행 및 확인](#6-실행-및-확인)
7. [서브도메인 추가하기](#7-서브도메인-추가하기)
8. [자주 하는 실수 / FAQ](#8-자주-하는-실수--faq)

---

## 1. 이 가이드에서 만들 것

이 가이드를 완료하면 아래처럼 **포트 번호 없이** 도메인으로 접속할 수 있습니다.

| 기존 방식 | 도메인 방식 |
|-----------|------------|
| `http://localhost:8090` | `http://myapp.local` |
| `http://localhost:8091` | `http://api.myapp.local` |

> **도메인 이름은 전부 자유롭게 바꿀 수 있습니다.**
> 이 가이드에서는 예시로 `myapp.local` 을 사용하지만, `myapp` 자리에 원하는 이름을 쓰면 됩니다.
> 예: `blog.local`, `shop.local`, `myproject.local`
>
> 단, 끝에 `.local` 은 그대로 유지하는 것을 권장합니다. `.local` 은 로컬 전용 도메인임을 나타내는 관례적인 접미사로, 실제 인터넷 도메인과 충돌하지 않습니다.

---

## 2. 전체 구조 이해하기

> **리버스 프록시란?**
> 브라우저가 보낸 요청을 대신 받아서 알맞은 서버로 전달해주는 중간 관리자입니다.
> 도메인 주소를 보고 "이건 앱 서버로, 저건 API 서버로" 하고 나눠줍니다.

```
브라우저
  │
  │  http://myapp.local        (80번 포트)
  │  http://api.myapp.local    (80번 포트)
  │
  ▼
Nginx 컨테이너 ← 리버스 프록시 역할
  │
  ├─── myapp.local        ──▶  php 컨테이너
  └─── api.myapp.local    ──▶  api 컨테이너
```

핵심은 **Nginx 하나가 80번 포트를 담당**하고, 도메인 이름을 보고 요청을 각 컨테이너에 나눠주는 것입니다.

---

## 3. Windows hosts 파일 수정

> **hosts 파일이란?**
> 도메인 이름을 IP 주소로 바꿔주는 파일입니다.
> 원래는 DNS 서버가 이 역할을 하지만, hosts 파일에 직접 적으면 DNS 서버를 거치지 않고 내 PC에서 바로 처리합니다.
> 쉽게 말해 "브라우저야, `myapp.local`은 내 컴퓨터(127.0.0.1)에 있어" 라고 알려주는 것입니다.

### 3-1. 메모장을 관리자 권한으로 실행

시작 메뉴에서 **메모장** 검색 → 우클릭 → **관리자 권한으로 실행**

> 관리자 권한이 없으면 저장이 안 됩니다.

### 3-2. hosts 파일 열기

메모장에서 **파일 → 열기** 후 경로 입력창에 아래 경로를 붙여넣습니다.

```
C:\Windows\System32\drivers\etc\hosts
```

> 파일 형식을 **모든 파일**로 바꿔야 hosts 파일이 보입니다.

### 3-3. 도메인 추가

파일 맨 아래에 아래 내용을 추가합니다.

```
127.0.0.1  myapp.local
```

> `127.0.0.1`은 내 컴퓨터를 가리키는 주소입니다.
> 위 내용은 "myapp.local 은 내 컴퓨터에 있다"는 뜻입니다.
>
> 서브도메인(`api.myapp.local` 등)을 추가하는 방법은 **7단계**에서 안내합니다.

### 3-4. 저장

**파일 → 저장** (Ctrl+S)

---

## 4. docker-compose.yml 수정

기존 docker-compose.yml 에 Nginx 컨테이너를 추가하고, 기존 컨테이너의 포트 설정을 변경합니다.

> **여기서 정하는 서비스 이름이 중요합니다**
> docker-compose.yml 의 각 서비스 이름(예: `app`, `api`, `admin`)은 Docker 내부에서 컨테이너끼리 서로를 부르는 이름이 됩니다.
> 이 이름을 5단계의 Nginx 설정에서 그대로 사용합니다.

### 4-1. 프로젝트 폴더를 IDE로 열기

Ubuntu 터미널에서 프로젝트 폴더를 IDE로 엽니다.

**VSCode를 사용하는 경우:**

```bash
cd ~/projects/ci3-sample
code .
```

VSCode가 열리면 왼쪽 파일 탐색기에서 `docker-compose.yml` 을 클릭합니다.

**PhpStorm을 사용하는 경우:**

이미 프로젝트가 열려 있다면 왼쪽 파일 탐색기에서 `docker-compose.yml` 을 더블클릭합니다.

### 4-2. 수정 내용

> 실제로 바꿀 부분은 딱 두 가지입니다.
> 1. 맨 위에 `nginx` 서비스 블록 추가
> 2. `php` 서비스의 `ports` → `expose` 로 변경
>
> `mysql` 서비스와 하단 `volumes:` 블록은 손대지 않습니다.

**변경 전:**

```yaml
services:

  mysql:
    image: mysql:8.0
    restart: unless-stopped
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci
    environment:
      MYSQL_ROOT_PASSWORD: root1234
      MYSQL_DATABASE: ci3_sample
      MYSQL_USER: ci3user
      MYSQL_PASSWORD: ci3pass
    volumes:
      - mysql_data:/var/lib/mysql
      - ./docker/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "3310:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot1234"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  php:
    build:
      context: ./docker
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
    ports:
      - "8090:80"          # ← 이 부분을 변경합니다
    volumes:
      - .:/var/www/html
    environment:
      DB_HOST: mysql
      DB_USER: ci3user
      DB_PASS: ci3pass
      DB_NAME: ci3_sample

volumes:
  mysql_data:
```

**변경 후:**

```yaml
services:

  nginx:                   # ← 이 블록을 새로 추가합니다
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
    depends_on:
      - php

  mysql:
    image: mysql:8.0
    restart: unless-stopped
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci
    environment:
      MYSQL_ROOT_PASSWORD: root1234
      MYSQL_DATABASE: ci3_sample
      MYSQL_USER: ci3user
      MYSQL_PASSWORD: ci3pass
    volumes:
      - mysql_data:/var/lib/mysql
      - ./docker/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "3310:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot1234"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  php:
    build:
      context: ./docker
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
    expose:
      - "80"               # ← ports 를 expose 로 변경
    volumes:
      - .:/var/www/html
    environment:
      DB_HOST: mysql
      DB_USER: ci3user
      DB_PASS: ci3pass
      DB_NAME: ci3_sample

volumes:
  mysql_data:
```

> **`ports` vs `expose` 차이**
> - `ports`: 내 PC에서도 접속 가능하도록 포트를 외부에 공개
> - `expose`: Docker 컨테이너끼리만 통신 가능, 외부에서는 접속 불가
>
> Nginx가 중간에서 요청을 받아주므로, php 컨테이너는 외부에 직접 노출할 필요가 없습니다.
>
> `mysql` 은 DB이므로 Nginx 와 무관합니다. 그대로 두면 됩니다.

### 4-3. 저장

`Ctrl+S` 로 저장합니다.

---

## 5. Nginx 리버스 프록시 설정

docker-compose.yml 에서 정한 서비스 이름을 여기서 사용합니다.
Nginx가 도메인을 보고 어느 컨테이너로 요청을 보낼지 알 수 있도록 설정 파일을 작성합니다.

### 5-1. 설정 파일 폴더 만들기

Ubuntu 터미널에서 Nginx 설정 폴더를 만듭니다.

```bash
cd ~/projects/ci3-sample
mkdir -p nginx/conf.d
```

### 5-2. Nginx 설정 파일 작성

IDE의 파일 탐색기에서 방금 만든 `nginx/conf.d/` 폴더 안에 `myapp.conf` 파일을 새로 만듭니다.

**VSCode를 사용하는 경우:**
`nginx/conf.d` 폴더에 마우스 우클릭 → **새 파일** → `myapp.conf` 입력 후 Enter

**PhpStorm을 사용하는 경우:**
`nginx/conf.d` 폴더에 마우스 우클릭 → **New → File** → `myapp.conf` 입력 후 Enter

아래 내용을 입력합니다. `proxy_pass` 뒤의 이름이 **4단계에서 정한 서비스 이름**과 일치해야 합니다.

```nginx
server {
    listen 80;
    server_name myapp.local;

    location / {
        proxy_pass http://php:80;   # docker-compose.yml 의 서비스 이름 'php'
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

> `proxy_pass http://php:80` 에서 `php` 부분이 docker-compose.yml 의 서비스 이름입니다.
> 다른 프로젝트에서 서비스 이름이 `app` 이라면 `http://app:80` 으로 써야 합니다.

`Ctrl+S` 로 저장합니다.

---

## 6. 실행 및 확인

### 6-1. 컨테이너 재시작

```bash
docker compose down
docker compose up -d
```

### 6-2. 컨테이너 상태 확인

```bash
docker ps
```

ci3-sample 기준으로 `nginx`, `php`, `mysql` 세 컨테이너가 모두 `Up` 상태여야 합니다.

### 6-3. 브라우저에서 접속

브라우저를 열고 아래 주소로 접속합니다.

```
http://myapp.local
```

기존에 `http://localhost:8090` 에서 보이던 화면이 그대로 표시되면 성공입니다.

---

## 7. 서브도메인 추가하기

`api.myapp.local` 을 추가한다고 가정합니다.
**연결할 컨테이너가 이미 있는지 없는지**에 따라 방법이 달라집니다.

---

### 경우 A. 연결할 컨테이너가 이미 실행 중인 경우

hosts 파일과 Nginx 설정만 추가하면 됩니다.

**A-1. hosts 파일에 도메인 추가**

`C:\Windows\System32\drivers\etc\hosts` 에 한 줄 추가:

```
127.0.0.1  api.myapp.local
```

**A-2. Nginx 설정 파일에 server 블록 추가**

`nginx/conf.d/myapp.conf` 파일에 아래 내용 추가:

```nginx
server {
    listen 80;
    server_name api.myapp.local;

    location / {
        proxy_pass http://php:80;   # 연결할 컨테이너의 서비스 이름
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**A-3. Nginx만 재시작**

```bash
docker compose restart nginx
```

> 컨테이너는 이미 실행 중이므로 Nginx만 재시작하면 됩니다.

---

### 경우 B. 연결할 컨테이너가 아직 없는 경우

docker-compose.yml 에 새 서비스를 먼저 추가한 뒤, hosts 파일과 Nginx 설정을 추가합니다.

**B-1. docker-compose.yml 에 새 서비스 추가**

```yaml
services:
  nginx:
    ...
    depends_on:
      - php
      - api        # ← 새로 추가

  php:
    expose:
      - "80"

  api:             # ← 새로 추가한 서비스
    image: ...
    expose:
      - "80"
```

**B-2. hosts 파일에 도메인 추가**

`C:\Windows\System32\drivers\etc\hosts` 에 한 줄 추가:

```
127.0.0.1  api.myapp.local
```

**B-3. Nginx 설정 파일에 server 블록 추가**

`nginx/conf.d/myapp.conf` 파일에 아래 내용 추가:

```nginx
server {
    listen 80;
    server_name api.myapp.local;

    location / {
        proxy_pass http://api:80;   # 새로 추가한 서비스 이름
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**B-4. 전체 컨테이너 재시작**

새 컨테이너가 추가됐으므로 전체를 다시 올려야 합니다.

```bash
docker compose down
docker compose up -d
```

---

## 8. 자주 하는 실수 / FAQ

### Q. 브라우저에서 도메인이 안 열린다

**hosts 파일 저장을 확인합니다.**
메모장을 관리자 권한으로 열지 않으면 저장이 실패합니다. 3-1 단계부터 다시 진행합니다.

**DNS 캐시를 초기화합니다.**
PowerShell에서 아래 명령 실행 후 브라우저를 새로 엽니다.

```powershell
ipconfig /flushdns
```

---

### Q. 80번 포트가 이미 사용 중이라는 에러가 난다

Windows에서 IIS나 다른 프로그램이 80번 포트를 점유 중인 경우입니다.
PowerShell에서 확인합니다.

```powershell
netstat -ano | findstr :80
```

IIS가 원인이라면 서비스를 중지합니다.

```powershell
net stop W3SVC
```

---

### Q. Nginx는 올라왔는데 앱 화면이 안 나온다

Nginx 로그를 확인합니다.

```bash
docker compose logs nginx
```

`proxy_pass` 에 적힌 컨테이너 이름이 docker-compose.yml 의 서비스 이름과 일치하는지 확인합니다.

---

### Q. 새로 추가한 서브도메인이 안 된다

Nginx 설정을 수정한 뒤 재시작을 빠뜨린 경우가 많습니다.

```bash
docker compose restart nginx
```

hosts 파일에도 해당 도메인을 추가했는지 확인합니다.

---

## 정리

| 해야 할 일 | 위치 |
|-----------|------|
| 도메인 등록 | Windows hosts 파일 |
| 도메인별 라우팅 설정 | `nginx/conf.d/myapp.conf` |
| 컨테이너 구성 | `docker-compose.yml` |
| 기존 컨테이너에 서브도메인 추가 시 | hosts 파일 + Nginx 설정 + `docker compose restart nginx` |
| 새 컨테이너와 함께 서브도메인 추가 시 | docker-compose.yml + hosts 파일 + Nginx 설정 + `docker compose down && up -d` |
