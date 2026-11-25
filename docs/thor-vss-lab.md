# Jetson AGX Thor에서 NVIDIA VSS 3.3.0 실습

**실습 목표**  
Thor에서 VSS `base` 프로필을 실행하고, 샘플 영상을 등록한 뒤 CLI와 NemoClaw 웹 채팅으로 영상에 질문한다. Windows에서는 SSH로 Thor를 관리하고, 같은 연결로 VSS 웹 UI를 연다.

명령 예시의 `LAB_WORK`는 실습 작업 폴더의 절대 경로로 설정한다. Windows SSH 예시에서는 `THOR_IP`, `thor-user`, 키 파일명을 실제 값으로 바꾼다.

## 1. Thor와 Docker 환경 확인

| 항목 | 실습 환경 |
| --- | --- |
| 기기 | NVIDIA Jetson AGX Thor Developer Kit, ARM64 |
| 운영체제 | Ubuntu 24.04, Jetson Linux 39.2.1 (JetPack 7.2.1 기반) |
| NVIDIA 드라이버 | 595.78 |
| Docker Engine / Compose / Buildx | 29.1.3 / v2.39.1 / 0.30.1 |
| NVIDIA Container Toolkit | 1.19.1 |
| VSS | v3.3.0, `base` 프로필, 빌드 이름 `thor-base-330` |

기존 Jetson Linux와 Docker를 사용했다. Compose가 빠져 있어 NVIDIA 문서의 ARM64 설치 방식으로 추가했다. Docker CLI 버전과 실제 데몬 버전은 아래처럼 각각 확인했다.

```bash
docker --version
sudo docker version --format '{{.Server.Version}}'
docker compose version
docker buildx version
nvidia-smi -L
```

Docker 컨테이너 안에서도 GPU가 보이는지 검사했다. 첫 실행에서 Ubuntu 이미지가 다운로드됐고, 출력에 `GPU 0: NVIDIA Thor`가 표시됐다.

```bash
sudo docker run --rm --runtime=nvidia --gpus all ubuntu:24.04 nvidia-smi -L
```

Docker의 cgroup 드라이버는 처음에 `systemd`였다. VSS 저장소를 받은 뒤 아래 공식 스크립트로 `cgroupfs`를 설정했다. 스크립트는 기존 설정을 백업하고 Docker를 재시작하므로, 실행 중인 컨테이너가 있다면 작업을 마친 뒤 실행한다.

```bash
cd "$LAB_WORK/video-search-and-summarization"
sudo bash deploy/docker/scripts/pin_docker_version.sh
sudo docker info --format 'Docker={{.ServerVersion}} cgroup={{.CgroupDriver}}'
# Docker=29.1.3 cgroup=cgroupfs
```

## 2. NGC 인증, 샘플 영상과 배포 패키지 준비

1. NVIDIA 문서의 ARM64 **NGC CLI 4.10.0**을 작업 폴더에 풀고 버전을 확인했다.
2. NGC 웹에서 개인용 조직을 선택하고 **NGC Catalog** 권한으로 **Personal API Key**를 발급했다.
3. `ngc config set`에서 출력 형식은 `ascii`, 조직은 키를 발급한 조직과 같은 항목을 선택했다. 개인 조직에서 팀과 ACE 선택지가 각각 `no-team`, `no-ace`뿐이어서 기본값을 사용했다.
4. VSS 3.3.0 샘플 아카이브를 내려받아 압축을 풀었다.

```bash
cd "$LAB_WORK"
./vss-tools/ngc-cli/ngc --version
./vss-tools/ngc-cli/ngc config set
./vss-tools/ngc-cli/ngc registry resource download-version \
  nvidia/vss-developer/dev-profile-sample-data:3.3.0
mkdir -p sample-data
tar -xf dev-profile-sample-data_v3.3.0/dev-profile-sample-data.tar.gz -C sample-data
```

원본 실습 영상은 아래 경로에 저장돼 있다.

```text
$LAB_WORK/sample-data/dev-profile-sample-data/warehouse_safety_0001.mp4
```

VSS 배포 패키지는 Git 저장소의 `v3.3.0` 태그를 사용했다. 체크아웃 이후 `git lfs pull`을 실행했고, LFS 포인터만 남은 파일이 없음을 확인했다.

```bash
# 새로 체크아웃할 때 실행. Git LFS가 PATH에 있어야 한다.
git clone --branch v3.3.0 https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization.git
cd video-search-and-summarization
git lfs install --local
git lfs pull
```

실습에서는 Git LFS도 작업 폴더에 풀어 사용했으므로, 기존 체크아웃에서 필요하면 `export PATH="$PWD/../vss-tools/usr/bin:$PATH"`를 실행한다.

## 3. Codex 배포 전 점검과 VSS CLI 실습

Quickstart의 **Prepare your coding agent** 단계에서 `uv`를 설치하고, 저장소의 `vss-build-vision-ai` 및 `skills/operations` 아래 스킬을 `~/.codex/skills`에 심볼릭 링크로 등록했다. 배포 전에는 Docker 접근 권한, 드라이버, 메모리, 디스크, 포트와 Thor 전용 런타임 설정을 점검했다. 새 호스트에서 재현할 때는 NVIDIA 사전 요구 사항의 전원 모드, 클럭, 커널 설정, 캐시 정리 절차도 확인한다.

샘플을 받는 NGC CLI 인증과 컨테이너 이미지를 받는 Docker 레지스트리 인증은 각각 진행했다. Thor에서 `docker info`가 성공하는 로컬 터미널을 사용했다. 계정을 `docker` 그룹에 추가한 경우 새 로그인 세션이나 `newgrp docker`로 그룹 변경을 적용한다. 이 그룹은 관리자 수준의 Docker 접근을 허용한다.

```bash
read -rsp 'NGC Personal API Key: ' NGC_CLI_API_KEY
printf '\n'
export NGC_CLI_API_KEY
printf '%s' "$NGC_CLI_API_KEY" |
  docker login nvcr.io --username '$oauthtoken' --password-stdin

cd "$LAB_WORK/video-search-and-summarization"
/usr/lib/chatgpt/resources/codex --no-daemon
```

이 기기의 앱 번들 CLI는 일반 `codex` 명령으로 검색되지 않았다. 번들 실행 파일을 직접 실행할 때 `--no-daemon`을 지정해 로컬 세션을 열었다.

**Deploy the base profile** 단계에서는 먼저 NemoClaw 없이 배포했다. VSS 웹 UI와 영상 입력·저장 서비스(VIOS/VST)를 띄우고, **Cosmos3 Nano Reasoner** 영상 모델은 Thor의 RT-VLM에서 실행했다. 배포 계획을 확인한 뒤 컨테이너를 시작했고, 이어서 샘플을 등록해 자연어 질문을 테스트했다.

아래는 같은 테스트를 재현하는 명령이다. `vss`가 PATH에 없으면 각 명령의 `vss`를 `uv run --project libs/vss vss`로 바꿔 저장소 루트에서 실행한다.

```bash
vss configure --base-url http://localhost:7777
vss vios list
# 목록에 없을 때 한 번 등록한다.
vss vios add --type video ../sample-data/dev-profile-sample-data/warehouse_safety_0001.mp4
vss vios list
vss vlm run --sensor warehouse_safety_0001 \
  --prompt '영상 속 작업자가 안전모를 쓰고 있나요? 보이는 근거를 설명해 주세요.'
```

**실습 결과**  
모델은 작업자가 안전모를 착용하고 있다고 응답했다. 영상은 [VSS 웹 UI](http://localhost:7777/ko)의 **Video Management**에서 확인할 수 있다.

## 4. 기존 VSS에 NemoClaw 웹 채팅 연결

CLI 실습 이후 기존 `thor-base-330` 배포에 NemoClaw를 추가했다. 구성 파일에서 **OpenClaw 웹 채팅 어댑터 활성화**와 **내부 릴레이 `18790` 설정**을 확인했다. 브라우저는 VSS의 채팅 사이드바를 사용하고, NemoClaw는 자연어 질문을 받아 VSS 도구를 호출한다. 영상 분석은 Thor의 RT-VLM이 수행한다.

대화 모델로 안내한 구성은 **NVIDIA Build 호스팅 Nemotron 3.5 Lightning** (`nvidia/nemotron-3.5-lightning-30b-a3b`)이다. 이 경로의 키는 **NVIDIA Build API Key**이며 `NVIDIA_API_KEY`로 전달한다. NGC Personal API Key는 이미지와 아티팩트 다운로드에 사용한다.

```bash
read -rsp 'NVIDIA Build API Key: ' NVIDIA_API_KEY
printf '\n'
export NVIDIA_API_KEY
```

웹 채팅 연결에 쓰는 **게이트웨이 토큰**은 아래 명령으로 Thor에서 조회한다. 토큰을 VSS 웹 UI의 **Connect NemoClaw chat**에 입력하고, **Video Management**에서 영상 재생과 채팅 사이드바를 함께 사용한다. API 키와 게이트웨이 토큰은 터미널과 연결 화면에만 입력한다.

```bash
nemoclaw thor-base-330 gateway-token --quiet
```

## 5. Windows와 VS Code에서 접속

Windows에서 만든 `thor-user.pub`의 내용을 Thor 계정의 `~/.ssh/authorized_keys`에 추가했다. 기존 키는 보존하고, `~/.ssh`와 `authorized_keys`의 권한을 각각 `700`과 `600`으로 설정한다. 개인키 `thor-user`은 Windows에 보관한다.

처음 `ssh thor`에서 호스트 이름 해석 오류가 나면 `thor` 별칭 설정을 확인한다. 공개키를 먼저 등록할 때는 PowerShell에서 Thor의 IP로 직접 접속할 수 있다. 아래 `THOR_IP`를 해당 IP로 바꾼다.

```powershell
Get-Content "$env:USERPROFILE\.ssh\thor-user.pub" |
  ssh thor-user@THOR_IP 'umask 077; mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

Windows의 `%USERPROFILE%\.ssh\config`에 아래 내용을 저장한다. `HostName`은 실습 당시 Thor의 Wi-Fi IP다. DHCP로 주소가 바뀌면 갱신한다.

```sshconfig
Host thor
    HostName THOR_IP
    User thor-user
    IdentityFile ~/.ssh/thor-user
    IdentitiesOnly yes
    LocalForward 127.0.0.1:7777 127.0.0.1:7777
```

```powershell
ssh -G thor | Select-String '^hostname '
ssh -o PreferredAuthentications=publickey thor
```

PowerShell에서 `ssh thor`로 연결하거나 VS Code에서 **Remote-SSH: Connect to Host… → thor**를 선택하면 된다. 연결을 유지한 상태에서 Windows 브라우저로 [VSS 웹 UI](http://localhost:7777/ko)를 연다. 영상과 VSS 안의 NemoClaw 채팅은 **포트 7777**로 접속한다. NemoClaw 자체 대시보드를 따로 열 때는 기본 `18789`를 추가 포워딩할 수 있다. `18790`은 Thor 내부에서 VSS와 NemoClaw를 연결하는 데 쓰인다.

## 참고 문서

- [NVIDIA VSS Quickstart](https://docs.nvidia.com/vss/getting-started/quickstart)
- [NVIDIA VSS 사전 요구 사항](https://docs.nvidia.com/vss/getting-started/prerequisites)
- [NVIDIA Container Toolkit GPU 테스트](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/sample-workload.html)
- [NVIDIA Build Nemotron 모델 및 API 키](https://build.nvidia.com/nvidia/nemotron-3.5-lightning-30b-a3b/build)
