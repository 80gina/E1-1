# 개발자 작업실 구축 미션 — 숲해설 안내 페이지 컨테이너화

## 1) 프로젝트 개요
터미널, Docker, Git/GitHub로 개발 작업실을 직접 구축하고 모든 과정을 증거로 남긴다.
실습 소재는 업무와 연결해 **'숲해설 프로그램 안내' 정적 웹페이지**를 nginx 컨테이너로 실행한다.

## 2) 실행 환경
- OS / Shell / 터미널: Windows 11 Home / PowerShell 7.6.9 / Windows Terminal
- Docker 버전: Docker version 29.9.2, build a82e528 (Docker Desktop, WSL2 백엔드)
- Git 버전: git version 2.53.0.windows.3

## 3) 수행 체크리스트
- [x] 터미널 기본 조작
- [x] 권한 실습 (파일 1, 디렉토리 1)
- [x] Docker 설치/점검
- [x] hello-world / ubuntu 실행
- [x] 커스텀 이미지 빌드·실행
- [x] 포트 매핑 접속 (8080, 8081)
- [x] 바인드 마운트 반영
- [x] 볼륨 영속성
- [x] Git 설정 + GitHub/VSCode 연동

## 4) 수행 로그와 검증 방법
각 항목은 `목적 → 명령어 → 출력 → 스크린샷` 순서로 기록한다.

### 4.1 터미널 조작
목적: 위치 확인, 목록(숨김 포함), 이동, 생성, 복사, 이름변경, 삭제, 내용 확인 (PowerShell 7.6.9)

```powershell
PS> cd C:\Users\yello\코디세이\sns\workstation
PS> pwd
C:\Users\yello\코디세이\sns\workstation
PS> ls -Force
d----   site
-a---   Dockerfile
-a---   README.md

PS> mkdir practice
PS> cd practice
PS> New-Item a.txt            # 빈 파일 생성 (리눅스 touch)
PS> echo "hello" > b.txt      # 내용이 있는 파일 생성
PS> ls -Force
-a---   0  a.txt
-a---   7  b.txt
PS> cat b.txt
hello

PS> cp b.txt c.txt            # 복사
PS> mv c.txt d.txt            # 이름 변경
PS> mkdir sub
PS> mv d.txt sub/             # 이동
PS> rm a.txt                  # 삭제
PS> ls -Recurse
practice:      sub, b.txt
practice\sub:  d.txt
```
- 증거: `screenshots/01_pwd_ls.png`, `03_create_files.png`, `04_copy_move_delete.png`

### 4.2 권한 실습
목적: 변경 전/후 비교. Windows에는 `chmod`가 없어 `attrib`(파일), `icacls`(디렉토리)로 수행했다.
리눅스 `chmod 755/644/700` 실습은 4.3의 ubuntu 컨테이너에서 별도로 수행한다.

**파일 (b.txt): 읽기 전용 설정**
```powershell
PS> attrib b.txt
A                  ...\practice\b.txt
PS> attrib +r b.txt
PS> attrib b.txt
A    R             ...\practice\b.txt
PS> echo "new" > b.txt
Out-File: Access to the path '...\practice\b.txt' is denied.
PS> attrib -r b.txt           # 복구
```
- 해석: 변경 후 `R`(읽기 전용) 속성이 붙었고, 쓰기가 거부되어 권한이 실제로 적용됨을 확인했다.

**디렉토리 (sub): 쓰기(WD) 거부 설정**
```powershell
PS> icacls sub /deny "${env:USERNAME}:(WD)"
PS> icacls sub
sub <계정>:(DENY)(WD)
    ... (상속 권한)
PS> New-Item sub\test.txt
New-Item: Access to the path '...\sub\test.txt' is denied.
PS> icacls sub /remove:d "${env:USERNAME}"   # 복구
PS> icacls sub                               # (DENY) 사라짐
```
- 해석: 변경 후 `(DENY)(WD)`가 추가되어 파일 생성이 거부됐고, 복구 후 원래 상태로 돌아왔다.
- 증거: `screenshots/05_file_permission.png`, `06_dir_permission.png`, `07_dir_restore.png`
- 리눅스 755 = rwxr-xr-x (소유자 전체, 그룹·기타 읽기·실행)

### 4.3 Docker 점검과 운영
**점검 (설치·데몬 동작)**
```powershell
PS> docker --version
Docker version 29.9.2, build a82e528
PS> docker info
(Client 정보 이후) Server: ... Kernel Version: 6.18.54.1-microsoft-standard-WSL2
OSType: linux / Architecture: aarch64 / CPUs: 12 / Total Memory: 7.549GiB
```
- 증거: `screenshots/11_docker_version.png`, `11_docker_info_ok.png`, `11a_docker_desktop_running.png`

**hello-world**
```powershell
PS> docker run hello-world
Unable to find image 'hello-world:latest' locally
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
PS> docker images
hello-world:latest   c567b231f656   22.6kB
PS> docker ps -a
d299819116f5   hello-world   "/hello"   Exited (0)
```
- 증거: `screenshots/14_hello_world.png`, `15_images_ps.png`
- 관찰: hello-world는 메시지를 출력하고 바로 종료되어 `ps`에는 안 보이고 `ps -a`에만 `Exited (0)`으로 보인다.

**ubuntu 실행과 내부 명령**
```powershell
PS> docker run -it --name u1 ubuntu bash
root@fd38a5f12b56:/# ls
bin boot dev etc home lib media mnt opt proc root run sbin srv sys tmp usr var
root@fd38a5f12b56:/# echo hi
hi
root@fd38a5f12b56:/# cat /etc/os-release     # Ubuntu 26.04.1 LTS
```
- 증거: `screenshots/16a_docker_inside_container_error.png`(컨테이너 안에서 docker 입력 시 오류), `17_chmod_linux.png`

**리눅스 권한 실습 (컨테이너 안, chmod)**
```bash
root@...:/tmp# ls -ld f.txt d
drwxr-xr-x 2 root root 4096 Nov  2 13:41 d
-rw-r--r-- 1 root root    6 Nov  2 13:41 f.txt
root@...:/tmp# chmod 644 f.txt
root@...:/tmp# chmod 700 d
root@...:/tmp# ls -ld f.txt d
drwx------ 2 root root 4096 Nov  2 13:41 d
-rw-r--r-- 1 root root    6 Nov  2 13:41 f.txt
root@...:/tmp# chmod 755 f.txt
root@...:/tmp# ls -l f.txt
-rwxr-xr-x 1 root root 6 Nov  2 13:41 f.txt
```
- 해석: 644 = rw-r--r--, 700 = rwx------, 755 = rwxr-xr-x. 숫자는 소유자·그룹·기타 순서로 r=4, w=2, x=1을 더한 값이다.
- 증거: `screenshots/17_chmod_linux.png`, `17a_chmod_exit.png`

**컨테이너 종료/유지 관찰 (run -it vs exec)**
```powershell
PS> docker run -d --name u2 ubuntu sleep infinity
PS> docker exec -it u2 bash
root@...:/# echo "exec로 들어옴"
root@...:/# exit
PS> docker ps -a
f2b33b2a5abd   ubuntu   "sleep infinity"   Up 3 minutes        u2
fd38a5f12b56   ubuntu   "bash"             Exited (0)          u1
d299819116f5   hello-world   "/hello"      Exited (0)          thirsty_wright
PS> docker stats --no-stream
u2   CPU 0.00%   MEM USAGE / LIMIT 3.434MiB / 7.549GiB   MEM 0.04%   PIDS 1
```
- 관찰: `run -it`로 들어간 `u1`은 셸이 컨테이너의 주 프로그램이라 `exit` 시 함께 종료(Exited)됐다. `exec -it`로 들어간 `u2`는 주 프로그램(`sleep infinity`)이 계속 돌아 `exit` 후에도 Up으로 유지됐다.
- 증거: `screenshots/18_ps_logs.png`, `18a_images_after_exit.png`, `19_exec_vs_run.png`, `20_docker_stats.png`
- 참고: PowerShell에서 `docker exec ... echo "한글"`을 입력하면 명령줄에 한글이 8진수 이스케이프로 보이지만 출력은 `exec로 들어옴`으로 정상이었다.

### 4.4 커스텀 이미지·포트 매핑
**이미지 빌드**
```powershell
PS> docker build -t forest-web:1.0 .
[+] Building 8.8s (7/7) FINISHED
 => [1/2] FROM docker.io/library/nginx:alpine
 => [2/2] COPY site/ /usr/share/nginx/html/
 => naming to docker.io/library/forest-web:1.0
PS> docker images
forest-web:1.0       712c678e99f9   93.2MB
hello-world:latest   c567b231f656   22.6kB
ubuntu:latest        6a06111c3b0a   181MB
```
- 증거: `screenshots/21_docker_build.png`

**포트 매핑: 같은 이미지로 컨테이너 2개 (호스트 8080, 8081 → 컨테이너 80)**
```powershell
PS> docker run -d -p 8080:80 --name forest-8080 forest-web:1.0
PS> docker run -d -p 8081:80 --name forest-8081 forest-web:1.0
PS> docker ps
forest-8081   forest-web:1.0   Up   0.0.0.0:8081->80/tcp
forest-8080   forest-web:1.0   Up   0.0.0.0:8080->80/tcp
PS> curl.exe http://localhost:8080
<h1>숲해설 프로그램 안내</h1>  (HTML 응답 확인)
PS> curl.exe http://localhost:8081
<h1>숲해설 프로그램 안내</h1>  (HTML 응답 확인)
```
- 브라우저 접속 (주소창에 포트 표시): `screenshots/23_browser_8080.png`, `screenshots/24_browser_8081.png`
- 터미널 증거: `screenshots/22_port_mapping.png`
- 포트 매핑이 필요한 이유: 컨테이너는 격리된 네트워크라 내부 포트(80)로 밖에서 바로 접속할 수 없어, 호스트 포트(8080/8081)를 컨테이너 포트에 연결해 준다. 호스트 포트를 다르게 하면 같은 이미지로 여러 컨테이너를 동시에 띄울 수 있다.

### 4.5 바인드 마운트
목적: 호스트(내 PC) 폴더를 컨테이너에 연결하면, 컨테이너를 다시 만들지 않아도 호스트에서 고친 내용이 바로 반영되는지 확인한다.
```powershell
PS> docker run -d -p 8082:80 --name bind-test -v "${PWD}\site:/usr/share/nginx/html" nginx:alpine

# 변경 전
PS> curl.exe http://localhost:8082
<h1>숲해설 프로그램 안내</h1>
<p>환경교육 현장 안내 페이지 (Docker nginx 실습)</p>

# 호스트에서 site\index.html 수정
PS> Set-Content -Path site\index.html -Encoding utf8 -Value '<!doctype html>...<h1>바인드 마운트로 바뀐 내용</h1>...'

# 변경 후 (컨테이너는 그대로)
PS> curl.exe http://localhost:8082
<h1>바인드 마운트로 바뀐 내용</h1>

# 원복
PS> git checkout -- site/index.html
PS> git status        # 변경 없음
```
- 해석: 컨테이너를 재생성하지 않았는데 응답이 바뀌었다. COPY는 빌드 시점 복사본이고, 바인드 마운트는 호스트 폴더를 그대로 공유하기 때문이다.
- 증거: `screenshots/27_bind_mount_before_after.png`

### 4.6 볼륨 영속성
목적: 컨테이너를 삭제해도 볼륨에 저장한 데이터가 유지되는지 확인한다.
```powershell
PS> docker volume create mydata
mydata
PS> docker volume ls
local     mydata
PS> docker run -d --name vol-test -v mydata:/data ubuntu sleep infinity
PS> docker exec vol-test bash -c "echo forest-note > /data/hello.txt && cat /data/hello.txt"
forest-note

# 컨테이너 삭제 전/후 비교
PS> docker rm -f vol-test
vol-test
PS> docker run -d --name vol-test2 -v mydata:/data ubuntu sleep infinity
PS> docker exec vol-test2 cat /data/hello.txt
forest-note
```
- 해석: 컨테이너 `vol-test`를 삭제한 뒤 새 컨테이너 `vol-test2`에 같은 볼륨을 연결했더니 `forest-note`가 그대로 남아 있었다. 컨테이너 안의 데이터는 컨테이너와 함께 사라지지만, 볼륨은 컨테이너와 독립적으로 유지된다.
- 증거: `screenshots/32_volume_persist.png`

### 4.7 Git 설정과 GitHub 연동
```powershell
PS> git config --global user.name
(이름)
PS> git config --global init.defaultBranch
main
PS> git config --list
...
init.defaultbranch=main
user.name=(마스킹)
user.email=(마스킹)
remote.origin.url=https://github.com/80gina/E1-1.git
branch.main.remote=origin
branch.main.merge=refs/heads/main
PS> git remote -v
origin  https://github.com/80gina/E1-1.git (fetch)
origin  https://github.com/80gina/E1-1.git (push)
```
- 설명: `git config --list`에는 `init.defaultbranch`가 `master`(시스템 기본)와 `main`(내 설정)으로 두 번 나오는데, 뒤에 읽힌 값(`main`)이 적용된다.
- 연동 방식: 로컬 폴더를 `git init`한 뒤 `git remote add origin`으로 GitHub 저장소를 연결하고, 원격의 기존 커밋은 `git fetch` + `git reset origin/main`으로 가져와 내 파일을 유지한 채 이어서 커밋했다.
- 증거: `screenshots/02_git_remote_status.png`, `33_git_config.png`(이메일 마스킹), `34_git_remote.png`, `08_first_push.png`
- VSCode GitHub 로그인: `screenshots/35_vscode_github.png`
- VSCode 저장소 연동(`main` 브랜치 표시): `screenshots/36_vscode_source_control.png`, `36a_vscode_explorer.png`

## 5) 커스텀 이미지 설명
- 선택한 방식: (A) 웹 서버 베이스 이미지 활용 + 정적 콘텐츠 교체
- 베이스: `nginx:alpine`
- 커스텀 포인트와 목적
  | 지시어 | 내용 | 목적 |
  |---|---|---|
  | `LABEL org.opencontainers.image.title="forest-guide-web"` | 이미지 이름표 | 이미지가 무엇인지 메타데이터로 식별 |
  | `ENV APP_ENV=dev` | 환경 구분 값 | 개발/운영 환경 구분에 쓰는 설정을 코드와 분리 |
  | `COPY site/ /usr/share/nginx/html/` | 내 안내 페이지로 교체 | 기본 nginx 페이지 대신 숲해설 안내 페이지 제공 |
- 빌드/실행: 4.4 참고 (`docker build -t forest-web:1.0 .`, `docker run -d -p 8080:80 ...`)

## 6) 트러블슈팅 (2건 이상)
### 문제 1: `docker info`에서 `failed to connect to the docker API`
- 문제: Client 정보는 나오지만 Server 부분에서 `open //./pipe/docker_engine: The system cannot find the file specified` 오류가 발생했다.
- 원인 가설: Docker 프로그램은 설치되어 있으나 엔진(Docker Desktop)이 실행되지 않았다.
- 확인 방법: `docker info`의 Client는 정상이고 Server만 실패함을 확인. Docker Desktop 앱을 실행해 본다.
- 해결: Docker Desktop을 실행하고 엔진이 Running이 된 뒤 다시 확인한다. (증거: `screenshots/10_docker_engine_off.png`)

### 문제 2: Docker Desktop 실행 시 `WSL not installed`
- 문제: Docker Desktop 실행 직후 "Windows Subsystem for Linux (WSL) is not installed" 오류 창이 떴다.
- 원인 가설: Windows에서 컨테이너는 리눅스 환경(WSL2) 위에서 동작하는데 WSL이 설치되어 있지 않았다.
- 확인 방법: 오류 메시지 안내에 따라 관리자 PowerShell에서 `wsl --install` 실행.
- 해결: `wsl --install` 후 재부팅하고 Docker Desktop을 다시 실행한다. (증거: `screenshots/12_wsl_not_installed.png`, `13_wsl_install.png`)

### 문제 3: 브라우저에서 `localhost:8080` 접속 시 `ERR_CONNECTION_REFUSED`
- 문제: 컨테이너를 실행한 뒤 시간이 지나 브라우저로 접속하니 "사이트에 연결할 수 없음(연결 거부)"이 떴다.
- 원인 가설: Docker 엔진이 꺼지면서 컨테이너도 함께 멈춰 해당 포트를 여는 프로세스가 없다.
- 확인 방법: `docker ps -a`에서 `forest-8080`, `forest-8081`이 `Exited (255)`인 것을 확인했다. (엔진 종료 시 비정상 종료 코드 255)
- 해결: Docker Desktop을 실행해 엔진을 켠 뒤 `docker start forest-8080 forest-8081`로 컨테이너를 다시 시작하고, 브라우저 접속이 되는 것을 확인했다. 이미지와 컨테이너는 삭제되지 않고 멈춰 있을 뿐이었다.
- 증거: `screenshots/25_connection_refused.png`, `26_docker_start_ok.png`, `23_browser_8080.png`

### 문제 4: 컨테이너 안에서 `docker` 명령 입력 시 `command not found`
- 문제: ubuntu 컨테이너 안(`root@...:/#`)에서 `docker run ...`을 입력하자 `bash: docker: command not found`가 나왔다.
- 원인 가설: `docker` CLI는 호스트(Windows)에 설치되어 있고, 컨테이너 안에는 없다.
- 확인 방법: 프롬프트가 `root@...`(컨테이너 안)임을 확인했다.
- 해결: `exit`로 컨테이너를 나와 호스트 PowerShell에서 docker 명령을 입력한다. (증거: `screenshots/16a_docker_inside_container_error.png`)

### 문제 5: `git checkout`에서 `fatal: not a git repository`
- 문제: 바인드 마운트 실습 후 `git checkout -- site/index.html`을 실행하자 `fatal: not a git repository (or any of the parent directories): .git`이 나왔다.
- 원인 가설: Git 저장소가 아닌 상위 폴더(`...\sns`)에서 명령을 실행했다. (터미널이 `sns`에서 시작됨)
- 확인 방법: 프롬프트의 경로가 `...\sns>`임을 확인했다. 저장소는 `...\sns\workstation`이다.
- 해결: `cd C:\Users\yello\코디세이\sns\workstation`으로 이동한 뒤 다시 실행했고, `git status`가 깨끗해졌다. (증거: `screenshots/30_not_a_git_repository.png`, `27_bind_mount_before_after.png`)

### 문제 6: `docker compose ps`에서 `no configuration file provided: not found` / `no such service: helper`
- 문제: 보너스 실습 중 `docker compose ps`가 설정 파일이 없다고 했고, 다른 폴더에서 `docker compose start helper`를 하자 `no such service: helper`가 나왔다.
- 원인 가설: `docker compose`는 **현재 폴더의 `docker-compose.yml`**을 읽는데, 파일이 없는 폴더(`workstation`)이거나 다른 프로젝트의 폴더(`compose-basic`)에서 실행했다. 서비스 이름(`helper`)은 그 폴더의 파일에 적힌 것만 유효하다.
- 확인 방법: 프롬프트 경로를 확인했다. 파일은 `bonus\compose-basic`, `bonus\compose-multi`에 각각 있다.
- 해결: 해당 폴더로 `cd`한 뒤 다시 실행했다. (`-f` 옵션으로 파일 경로를 지정해도 된다.)
- 증거: `screenshots/42_compose_no_config_fix.png`, `40_compose_basic.png`, `48_compose_ops_stop_start.png`

## 6-1) 보너스 과제 (Docker Compose, 환경 변수)
파일: `bonus/compose-basic/docker-compose.yml`, `bonus/compose-multi/docker-compose.yml`, `bonus/compose-multi/.env`

### 보너스 1. Docker Compose 기초 (단일 서비스)
`docker run -p 8090:80 -v ... nginx:alpine` 같은 긴 명령을 파일로 문서화했다.
```powershell
PS ...\bonus\compose-basic> cat docker-compose.yml
services:
  web:
    image: nginx:alpine
    container_name: compose-web
    ports:
      - "8090:80"
    volumes:
      - ../../site:/usr/share/nginx/html:ro
PS> docker compose up -d
 ✔ Network compose-basic_default  Created
 ✔ Container compose-web          Started
PS> docker compose ps          # compose-web  Up  0.0.0.0:8090->80/tcp
PS> curl.exe http://localhost:8090   # 숲해설 프로그램 안내
PS> docker compose down
 ✔ Container compose-web Removed
 ✔ Network compose-basic_default Removed
```
- 배움: 실행 명령이 "문서화된 실행 설정"이 되어 같은 환경을 누구나 같은 명령(`docker compose up -d`)으로 재현할 수 있다.
- 증거: `screenshots/40_compose_basic.png`, `41_compose_basic_browser.png`, `42_compose_no_config_fix.png`, `43_compose_down.png`

### 보너스 2. Compose 멀티 컨테이너 (web + helper)
```powershell
PS ...\bonus\compose-multi> docker compose up -d
 ✔ Image alpine:3.20                  Pulled
 ✔ Network compose-multi_default      Created
 ✔ Container compose-multi-web        Started
 ✔ Container compose-multi-helper     Started
PS> docker compose exec helper wget -qO- http://web
<h1>숲해설 프로그램 안내</h1>          # 서비스 이름(web)으로 통신 성공
PS> docker compose exec helper wget -qO- http://localhost
wget: can't connect to remote host: Connection refused   # helper의 localhost는 자기 자신
```
- 배움: Compose가 같은 네트워크를 자동으로 만들고, **서비스 이름이 곧 주소**가 된다(서비스 디스커버리). 컨테이너마다 `localhost`는 자기 자신이라 다른 컨테이너는 서비스 이름으로 부른다.
- 증거: `screenshots/44_compose_multi_up.png`, `45_compose_multi_network.png`

### 보너스 3. Compose 운영 명령어 (up / ps / logs / down)
```powershell
PS> docker compose logs web --tail 5
compose-multi-web | 172.18.0.3 ... "GET / HTTP/1.1" 200 252 "-" "Wget"
compose-multi-web | 172.18.0.1 ... "GET / HTTP/1.1" 200 252 "-" "curl/8.22.0"
PS> docker compose stop helper     # helper만 Stopped
PS> docker compose ps -a           # helper Exited (137), web Up
PS> docker compose start helper    # helper Started
PS> docker compose ps              # 둘 다 Up
PS> docker compose down            # 컨테이너와 네트워크 정리
```
- 배움: `up → ps → logs → down`을 상태 확인 루틴으로 쓰면 어느 서비스가 살아 있고 누가 요청했는지 바로 알 수 있다. 로그에는 요청한 쪽(Wget, curl)까지 기록된다.
- 증거: `screenshots/47_compose_ops_logs.png`, `48_compose_ops_stop_start.png`, `49_compose_env_override.png`(down 포함)

### 보너스 4. 환경 변수 활용 (포트·모드 변경)
`docker-compose.yml`에 `"${HOST_PORT:-8091}:80"`, `APP_ENV=${APP_ENV:-dev}`로 변수를 꽂고, `.env`에 기본값(`HOST_PORT=8091`, `APP_ENV=dev`)을 두었다. 파일을 고치지 않고 터미널의 환경 변수로 바꿨다.
```powershell
PS> $env:HOST_PORT="8095"
PS> $env:APP_ENV="prod"
PS> docker compose up -d
PS> docker compose ps
compose-multi-web  forest-web:1.0  Up  0.0.0.0:8095->80/tcp     # 8091 -> 8095
PS> docker compose exec helper printenv APP_ENV
prod                                                            # dev -> prod
PS> curl.exe http://localhost:8095                               # 접속 성공
PS> docker compose down
PS> Remove-Item Env:HOST_PORT; Remove-Item Env:APP_ENV           # 임시 변수 정리
```
- 배움: 설정(포트·모드)과 코드(compose 파일)를 분리하면 같은 파일로 dev와 prod를 다르게 실행할 수 있다. 터미널 환경 변수가 `.env`보다 우선한다.
- 증거: `screenshots/49_compose_env_override.png`, `49a_compose_env_check.png`, `50_compose_env_cleanup.png`

### 보너스 5. GitHub SSH 키 설정
HTTPS 대신 SSH 키로 인증해 push했다. 키는 한 쌍이며, **공개키(`.pub`)만 GitHub에 등록**하고 **개인키는 내 PC의 `.ssh` 폴더에만** 둔다(저장소, 문서, 스크린샷에 포함하지 않음).
```powershell
PS> ssh -V
OpenSSH_for_Windows_9.5p2
PS> ssh-keygen -t ed25519 -C "github-ssh-e1"     # 키 한 쌍 생성 (기본 위치 ~\.ssh\id_ed25519)
Your public key has been saved in C:\Users\yello/.ssh/id_ed25519.pub
SHA256:K8fl5VUVOv/5w/QzirR++... github-ssh-e1
PS> Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard   # 공개키만 복사
# GitHub → Settings → SSH and GPG keys → New SSH key 에 붙여넣기 (Title: 내 윈도우 PC)

PS> ssh -T git@github.com
Hi 80gina! You've successfully authenticated, but GitHub does not provide shell access.

PS> git remote set-url origin git@github.com:80gina/E1-1.git
PS> git remote -v
origin  git@github.com:80gina/E1-1.git (fetch)
origin  git@github.com:80gina/E1-1.git (push)
PS> git push                                      # SSH로 push 성공 (스크린샷 55)
```
- GitHub에 등록된 키의 지문(`SHA256:K8fl5VUVOv...`)이 터미널에서 만든 키의 지문과 일치함을 확인했다.
- 처음 접속할 때 GitHub 서버의 지문(`SHA256:+DiY3wvv...`)이 공식 지문과 같음을 확인하고 `yes`로 신뢰 목록에 추가했다.
- 배움: HTTPS는 로그인 창·토큰으로 인증하고, SSH는 키 쌍으로 인증한다. 공개키(자물쇠)는 공유해도 되지만 개인키(열쇠)는 절대 공유하지 않는다.
- 증거: `screenshots/51_ssh_keygen.png`, `52_github_ssh_key_added.png`, `53_ssh_test.png`, `54_ssh_remote.png`, `55_ssh_push.png`

## 7) 학습 정리
- **절대/상대 경로:** 절대 경로는 드라이브 맨 앞(`C:\` 또는 `/`)부터 쓴 전체 주소(예: `C:\Users\yello\코디세이\sns\workstation`)이고, 상대 경로는 지금 위치 기준 주소(예: `site\index.html`, `..\`)다. 같은 파일도 어디서 실행하느냐에 따라 상대 경로가 달라진다.
- **권한 r/w/x와 755, 644:** r=읽기(4), w=쓰기(2), x=실행(1)이고, 숫자는 소유자·그룹·기타 순서로 더한 값이다. 755는 `rwxr-xr-x`(소유자 전체, 나머지 읽기·실행), 644는 `rw-r--r--`(소유자 읽기·쓰기, 나머지 읽기)이다. 4.3에서 `chmod 644/700/755`로 직접 확인했다.
- **커스텀 이미지:** 기존 `nginx:alpine` 이미지를 베이스로 `Dockerfile`에 `LABEL`, `ENV`, `COPY`를 더해 내 이미지(`forest-web:1.0`)를 만들었다.
- **포트 매핑이 필요한 이유:** 컨테이너는 격리된 네트워크라 내부 포트(80)로 밖에서 바로 접속할 수 없다. `-p 호스트:컨테이너`로 연결해야 브라우저로 접속할 수 있고, 호스트 포트를 다르게 하면 같은 이미지로 컨테이너를 여러 개 띄울 수 있다.
- **바인드 마운트 vs 볼륨:** 바인드 마운트는 내 PC 폴더를 그대로 공유해 수정이 즉시 반영된다(개발용). 볼륨은 Docker가 관리하는 저장소로 컨테이너를 삭제해도 데이터가 남는다(데이터 보존용).
- **볼륨(영속 데이터):** 컨테이너 안의 데이터는 컨테이너와 함께 사라지지만, 볼륨에 저장한 데이터는 컨테이너를 지워도 새 컨테이너에 연결해 다시 읽을 수 있다(4.6).
- **Git vs GitHub:** Git은 내 PC에서 변경 이력을 기록하는 로컬 버전 관리 도구이고, GitHub는 그 저장소를 인터넷에 올려 공유·협업하는 원격 플랫폼이다. 로컬에서 `commit`으로 기록하고 `push`로 GitHub에 올렸다.
- **배운 점:** 오류(엔진 꺼짐, WSL 미설치, 연결 거부, 작업 폴더 착오)를 겪고 원인을 확인해 해결하는 과정이 가장 큰 학습이었다. 6) 트러블슈팅 참고.
