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
- [ ] 커스텀 이미지 빌드·실행
- [ ] 포트 매핑 접속 (8080, 8081)
- [ ] 바인드 마운트 반영
- [ ] 볼륨 영속성
- [ ] Git 설정 + GitHub/VSCode 연동

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
스크린샷: `screenshots/port-8080.png`, `screenshots/port-8081.png` (주소창 포함)
```bash
docker build -t forest-web:1.0 .
docker run -d -p 8080:80 --name forest-8080 forest-web:1.0
docker run -d -p 8081:80 --name forest-8081 forest-web:1.0
curl http://localhost:8080
```

### 4.5 바인드 마운트
```bash
docker run -d -p 8082:80 --name bind-test -v "$(pwd)/site:/usr/share/nginx/html" nginx:alpine
curl http://localhost:8082
(site/index.html 수정)
curl http://localhost:8082
```

### 4.6 볼륨 영속성
```bash
docker volume create mydata
docker run -d --name vol-test -v mydata:/data ubuntu sleep infinity
docker exec vol-test bash -c "echo 조사기록 > /data/note.txt && cat /data/note.txt"
docker rm -f vol-test
docker run -d --name vol-test2 -v mydata:/data ubuntu sleep infinity
docker exec vol-test2 cat /data/note.txt
```

### 4.7 Git 설정과 GitHub 연동
```bash
git config --global user.name "(이름)"
git config --global user.email "(이메일)"
git config --global init.defaultBranch main
git config --list
```
- VSCode GitHub 로그인 캡처: `screenshots/vscode-github.png` (토큰·이메일 마스킹)

## 5) 커스텀 이미지 설명
- 베이스: `nginx:alpine`
- LABEL: 이미지 설명 / ENV: 환경 구분(dev) / COPY: 내 안내 페이지로 교체

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

## 7) 학습 정리
- 절대/상대 경로:
- 권한 r/w/x와 755, 644:
- 포트 매핑이 필요한 이유:
- 볼륨(영속 데이터):
- Git vs GitHub:
