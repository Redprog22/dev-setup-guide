# WSL + Docker 개발 환경 세팅 가이드

Windows에서 WSL2 + Docker + IDE(VSCode / PhpStorm)를 이용한 개발 환경 구성 가이드입니다.

---

## 목차

1. [사전 확인](#1-사전-확인)
2. [WSL2 설치](#2-wsl2-설치)
3. [Ubuntu 초기 설정](#3-ubuntu-초기-설정)
4. [Docker Engine 설치](#4-docker-engine-설치)
5. [IDE 연결](#5-ide-연결)
   - [VSCode](#51-vscode)
   - [PhpStorm](#52-phpstorm)
6. [프로젝트 세팅](#6-프로젝트-세팅)
7. [프로젝트 실행](#7-프로젝트-실행)
8. [자주 하는 실수 / FAQ](#8-자주-하는-실수--faq)

---

## 1. 사전 확인

WSL2는 **Windows 10 버전 2004 이상** 또는 **Windows 11**에서 동작합니다.

PowerShell을 열고 Windows 버전을 확인하세요.

```powershell
winver
```

- Windows 10 빌드 **19041** 이상 → 진행 가능
- 그 이하 → Windows Update 먼저 진행

---

## 2. WSL2 설치

### 2-1. PowerShell을 관리자 권한으로 실행

시작 메뉴에서 **PowerShell** 검색 → 우클릭 → **관리자 권한으로 실행**

### 2-2. WSL 설치 명령 실행

```powershell
wsl --install
```

> Ubuntu가 기본으로 함께 설치됩니다.

### 2-3. 재부팅

설치 완료 후 **컴퓨터를 재부팅**합니다.

### 2-4. WSL 버전 확인

재부팅 후 PowerShell에서 확인합니다.

```powershell
wsl --version
```

WSL 버전이 표시되면 정상입니다. 혹시 오래된 버전이면 업데이트합니다.

```powershell
wsl --update
```

---

## 3. Ubuntu 초기 설정

재부팅 후 **Ubuntu** 앱이 자동으로 실행됩니다. (없으면 시작 메뉴에서 Ubuntu 검색)

### 3-1. 사용자 계정 생성

처음 실행 시 아이디와 비밀번호를 설정합니다.

```
Enter new UNIX username: myname        ← 사용할 아이디 입력
New password: ****                     ← 비밀번호 입력 (화면에 표시 안 됨)
Retype new password: ****
```

### 3-2. 패키지 업데이트

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 4. Docker Engine 설치

> Docker Desktop이 아닌 **WSL 안에 직접 설치**하는 방법입니다.
> 이렇게 하면 Node.js, PHP, MySQL 등 언어나 DB를 WSL에 직접 설치하지 않아도 됩니다.
> Docker가 각각의 실행 환경을 격리해서 관리해주기 때문입니다.

Ubuntu는 `apt`라는 패키지 관리자를 통해 프로그램을 설치합니다.
Docker는 Ubuntu 기본 저장소에 포함되어 있지 않기 때문에, **Docker 공식 저장소를 먼저 등록**한 뒤 설치해야 합니다.
4-1 ~ 4-3 단계가 바로 그 등록 과정입니다.

### 4-1. 필수 패키지 설치

저장소 등록에 필요한 도구들을 먼저 설치합니다.

- `ca-certificates` : HTTPS 통신을 위한 인증서
- `curl` : URL에서 파일을 다운로드하는 도구
- `gnupg` : 파일의 서명을 검증하는 도구
- `lsb-release` : Ubuntu 버전 정보를 확인하는 도구

```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### 4-2. Docker 공식 GPG 키 추가

> **GPG 키란?**
> 인터넷에서 파일을 받을 때 "이 파일이 진짜 Docker가 만든 것인지" 검증하는 디지털 서명입니다.
> 이 키를 등록해두면 나중에 Docker를 설치하거나 업데이트할 때 위조된 파일이 설치되는 것을 막아줍니다.

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### 4-3. Docker 저장소 추가

> **저장소(Repository)란?**
> apt가 프로그램을 찾아서 설치하는 "출처 목록"입니다.
> 여기서는 Docker 공식 주소를 그 목록에 추가하는 작업입니다.
> 이 단계가 없으면 apt가 Docker를 어디서 받아야 할지 모릅니다.

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 4-4. Docker Engine 설치

저장소 등록이 완료됐으니 이제 실제로 Docker를 설치합니다.

- `docker-ce` : Docker 엔진 본체
- `docker-ce-cli` : 터미널에서 docker 명령을 사용하게 해주는 도구
- `containerd.io` : 컨테이너를 실제로 실행하는 런타임
- `docker-compose-plugin` : 여러 컨테이너를 한 번에 관리하는 `docker compose` 명령 추가

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 4-5. Docker 서비스 시작

설치 후 Docker 엔진을 실행합니다.

```bash
sudo service docker start
```

### 4-6. 현재 사용자를 docker 그룹에 추가

기본적으로 docker 명령은 관리자 권한(`sudo`)이 필요합니다.
아래 명령을 실행하면 매번 `sudo`를 붙이지 않아도 됩니다.

```bash
sudo usermod -aG docker $USER
```

> 터미널을 **완전히 닫고 다시 열어야** 적용됩니다.

### 4-7. 설치 확인

```bash
docker --version
docker compose version
```

두 명령 모두 버전이 출력되면 완료입니다.

### 4-8. WSL 시작 시 Docker 자동 실행 설정

WSL을 열 때마다 Docker를 수동으로 시작하지 않도록 설정합니다.
아래 명령은 터미널이 열릴 때마다 Docker를 자동으로 켜주는 한 줄을 설정 파일에 추가합니다.

```bash
echo 'sudo service docker start > /dev/null 2>&1' >> ~/.bashrc
```

> zsh를 사용하는 경우 `.bashrc` 대신 `.zshrc`에 추가합니다.

---

## 5. IDE 연결

### 5-1. VSCode

#### VSCode 설치

[https://code.visualstudio.com](https://code.visualstudio.com) 에서 Windows용 VSCode를 다운로드하여 설치합니다.

#### WSL 확장 설치

VSCode 실행 후 왼쪽 Extensions 아이콘 클릭 (또는 `Ctrl+Shift+X`) →
**`WSL`** 검색 → Microsoft의 **WSL** 확장 설치

#### WSL 프로젝트 열기

Ubuntu 터미널에서 프로젝트 폴더로 이동 후 아래 명령을 실행합니다.

```bash
cd ~/projects/my-project
code .
```

> 처음 실행 시 VSCode Server가 WSL 안에 자동으로 설치됩니다. 잠시 기다리면 VSCode가 WSL 모드로 열립니다.

VSCode 왼쪽 하단에 **`WSL: Ubuntu`** 표시가 보이면 정상적으로 연결된 것입니다.

---

### 5-2. PhpStorm

#### PhpStorm 설치

[https://www.jetbrains.com/phpstorm](https://www.jetbrains.com/phpstorm) 에서 Windows용 PhpStorm을 다운로드하여 설치합니다.

#### WSL 프로젝트 열기

1. PhpStorm 실행
2. 상단 메뉴 **File → Remote Development**
3. **WSL** 선택
4. **Ubuntu** 선택 후 **Next**
5. **Project directory** 에 WSL 경로 입력

```
/home/myname/projects/my-project
```

6. **Start IDE and Connect** 클릭

> 처음 연결 시 PhpStorm Backend가 WSL 안에 설치됩니다. 완료되면 WSL 안의 프로젝트가 열립니다.

---

## 6. 프로젝트 세팅

> **실습 예제**: 이 가이드에서는 [CI3 메모장 샘플](https://github.com/Redprog22/ci3-sample) 을 예제로 사용합니다.
> CodeIgniter 3 + MySQL + Docker로 구성된 간단한 메모 CRUD 프로젝트입니다.

### 6-1. 프로젝트 폴더 생성

Ubuntu 터미널에서 실행합니다.

```bash
mkdir -p ~/projects
cd ~/projects
```

### 6-2. GitHub에서 Clone

```bash
git clone https://github.com/Redprog22/ci3-sample.git
cd ci3-sample
```

### 6-3. 컨테이너 시작

```bash
docker compose up -d
```

처음 실행 시 Docker 이미지를 빌드하므로 1~2분 정도 걸립니다.
완료 후 브라우저에서 **http://localhost:8090** 으로 접속하면 바로 동작하는 것을 확인할 수 있습니다.

> `docker compose up -d` 한 줄로 PHP, MySQL, Apache가 전부 자동으로 실행됩니다.
> WSL에 PHP나 MySQL을 직접 설치할 필요가 없습니다.

### 6-4. PHP 의존성 설치 (다른 프로젝트에서 vendor 폴더가 없는 경우)

`vendor` 폴더가 없는 경우 Docker를 이용해 설치합니다.
WSL에 PHP나 Composer를 직접 설치할 필요가 없습니다.

```bash
# 프로젝트 루트에 composer.json이 있는 경우
docker run --rm -v $(pwd):/app -w /app composer:latest install --ignore-platform-reqs
```

### 6-4. Node.js 의존성 설치 (Node 프로젝트인 경우)

`node_modules` 폴더가 없고 Dockerfile에서 직접 설치하지 않는 경우에만 실행합니다.

```bash
docker run --rm -v $(pwd):/app -w /app node:20-alpine sh -c "npm install"
```

> 대부분의 프로젝트는 Dockerfile 안에서 `npm install`을 처리하므로 이 단계가 필요 없을 수 있습니다.
> 프로젝트의 `docker-compose.yml`에서 `build: .` 형태라면 건너뜁니다.

---

## 7. 프로젝트 실행

### 7-1. 컨테이너 시작

```bash
cd ~/projects/레포이름
docker compose up -d
```

### 7-2. 컨테이너 상태 확인

```bash
docker ps
```

모든 컨테이너가 `Up` 또는 `healthy` 상태이면 정상입니다.

### 7-3. 로그 확인

```bash
docker compose logs -f
```

### 7-4. 컨테이너 중지

```bash
docker compose down
```

---

## 8. 자주 하는 실수 / FAQ

### Q. WSL을 열 때마다 Docker가 안 된다

WSL은 재시작하면 Docker 서비스가 꺼집니다.

```bash
sudo service docker start
```

4-8 단계의 자동 시작 설정을 했다면 자동으로 실행됩니다.

---

### Q. 포트가 이미 사용 중이라는 에러가 난다

같은 포트를 사용하는 다른 컨테이너가 실행 중인 경우입니다.

```bash
# 실행 중인 컨테이너 목록 확인
docker ps

# 해당 프로젝트 컨테이너 전부 중지
cd ~/projects/다른프로젝트
docker compose down
```

---

### Q. WSL 재시작 후 컨테이너가 전부 꺼졌다

WSL을 종료하면 컨테이너도 함께 꺼집니다. 다시 시작해주세요.

```bash
cd ~/projects/레포이름
docker compose up -d
```

---

### Q. `code .` 명령이 안 된다

VSCode WSL 확장이 설치되지 않았거나 WSL 버전이 낮을 수 있습니다.

```powershell
# PowerShell에서 WSL 업데이트
wsl --update
```

업데이트 후 WSL을 재시작합니다.

```powershell
wsl --shutdown
```

---

### Q. Windows 경로(`C:\...`)에 프로젝트를 두면 안 되나?

가능하지만 **권장하지 않습니다.**
Windows 경로를 Docker에 mount하면 파일 I/O 속도가 매우 느립니다.
항상 WSL 경로(`~/projects/...`)에 프로젝트를 두세요.

---

## 정리

| 역할 | 위치 |
|------|------|
| 소스코드 | WSL (`~/projects/프로젝트명`) |
| 실행환경 (Node, PHP, DB 등) | Docker 컨테이너 |
| 코드 편집 | VSCode 또는 PhpStorm (WSL 연결) |
| 패키지 설치 | Docker 컨테이너 안에서 실행 |

---

## 다음 단계

로컬 도메인 및 서브도메인 설정 관련 가이드는 아래를 참고하세요.

→ [로컬 도메인 설정 가이드](./README-domain.md)
