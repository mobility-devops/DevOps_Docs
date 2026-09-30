# DevOps 프로젝트 서버 초기 구축

> 원본: 노션 「회의 기록 / 09-28 / DevOps 프로젝트 서버 초기 구축」에서 2026-09-30 기준으로 옮김.

> DevOps 프로젝트 실습 환경 구성을 위해 물리 PC에 Ubuntu Desktop을 설치하고 기본 OS·네트워크·원격 접속·Docker·개발 도구 환경을 구축했다.
> 이후 VirtualBox에 Ubuntu Server VM을 생성하고 NAT Network, Static IP, SSH 접속 환경까지 구성했다.

## 📦 회의 시간 전 준비 및 구축 계획

### 1. 설치 미디어 및 프로그램 사전 준비

프로젝트 실습 환경 구축 시간을 줄이기 위해 필요한 OS 이미지와 일부 설치 파일을 USB에 미리 준비했다.

#### SanDisk USB 64GB

```text
Ubuntu 24.04.5.1 Desktop 부팅 USB
└─ Rufus 4.15로 제작
   └─ GPT / UEFI(비 CSM) / Large FAT32
```

#### Samsung USB

```text
Ubuntu 24.04.5.1/
└─ ubuntu-24.04.5-live-server-amd64.iso → VM 설치용

code_1.139.1-1790309529_amd64.deb → VS Code 1.139.1
virtualbox-7.2_7.2.20-175154~Ubuntu~noble_amd64.deb → VirtualBox 7.2.20
idea-2026.2.3.tar.gz → IntelliJ IDEA 2026.2.3 (x86_64)
```

모든 파일은 공식 SHA256 해시와 비교하여 검증 완료했으며, 부팅 USB는 ISO 내부 `md5sum.txt`로 448개 파일 무결성 확인을 완료했다.

VS Code와 IntelliJ IDEA는 설치 시간 단축을 위해 USB에 미리 파일을 준비했고, Chrome·Postman·Notion은 Desktop 설치 후 인터넷으로 설치할 계획으로 준비했다.

### 2. Ubuntu Desktop 재설치 결정

기존 컴퓨터에 설치되어 있던 Ubuntu 환경을 그대로 프로젝트에 사용하는 경우 이전 설정·패키지·파일 등의 영향으로 이후 프로젝트 진행 시 불편할 가능성이 있다고 판단했다.

따라서 USB에 준비한 `ubuntu-24.04.5.1-desktop-amd64` 이미지를 이용해 **Ubuntu Desktop을 새 OS로 설치**하기로 했다.

### 3. Ubuntu 24.04 LTS 버전 선택 이유

#### LTS 장기 지원

- Ubuntu 24.04 LTS 기반 환경 사용
- 장기 지원 버전을 선택해 프로젝트 환경의 안정성을 우선

#### 안정성과 호환성

- 26.04 LTS가 나와 있지만 초기 안정화 기간을 고려
- VirtualBox 등 대부분의 실습 도구가 Ubuntu 24.04 `noble`용 패키지를 제공
- 최신 기능보다 프로젝트에서 사용할 도구의 안정성과 호환성을 우선

#### 최신 포인트 릴리스

- 출시 후 누적된 보안·버그 패치가 포함된 최신 포인트 릴리스를 사용
- 준비 당시 Desktop ISO는 `24.04.5.1` 이미지를 사용

### 4. 구축 전 계획

1. 현재 PC 사양 및 가상화 지원 여부 확인
2. Ubuntu 24.04.5.1 Desktop을 새 OS로 설치
3. Ubuntu Server 24.04.5 ISO를 이용해 VirtualBox VM 구성
4. Backend 개발 및 프로젝트 작업을 위한 IntelliJ IDEA, VS Code, Notion, Google Chrome, Postman 등 설치
5. 물리 서버 IP 고정
6. 원격 접속 작업을 위한 SSH 환경 구성

#### PC 사양 확인 명령

```bash
echo "== CPU 아키텍처 =="; uname -m
echo "== CPU =="; lscpu | grep -E "Model name|^CPU\(s\)"
echo "== 가상화 지원(0이면 BIOS에서 꺼짐) =="; egrep -c '(vmx|svm)' /proc/cpuinfo
echo "== 메모리 =="; free -h | grep Mem
echo "== 디스크 =="; lsblk -d -o NAME,SIZE,MODEL,TYPE | grep disk
echo "== 현재 Ubuntu 버전 =="; lsb_release -ds
```

필요한 경우 BIOS/UEFI 진입은 다음 명령 또는 부팅 시 `F2` / `Del` 키를 이용하도록 준비했다.

```bash
sudo systemctl reboot --firmware-setup
```

### 5. 진행 상황

실제 구축 결과는 아래 **Ubuntu GUI 설치 및 세팅 과정**과 **VM 설치 및 세팅 과정**에서 상세히 기록한다.

> ✅ **현재 완료된 범위**
>
> Ubuntu 24.04.5 LTS 물리 호스트의 시스템 업데이트·hostname·Asia/Seoul/NTP·GUI·한글 입력, Wi-Fi 및 Static IP `192.168.200.149`, Gateway/DNS/Internet, OpenSSH·UFW·외부 PC SSH, 기본 관리 도구, Docker Engine 29.8.1·containerd 2.3.6·Docker Compose v5.5.1·hello-world 검증, Chrome·IntelliJ IDEA 2026.2.3·VS Code·VirtualBox 7.2.20·Postman 설치까지 완료했다.

> 🧱 **VirtualBox / VM 진행 상황**
>
> 호스트 자원은 i5-14400 / 16 논리 CPU, RAM 62 GiB, 확인 당시 SSD 약 422 GB 여유 공간이다. `UbuntuNetwork` NAT Network `10.0.2.0/24`, Gateway `10.0.2.1`, Host `127.0.0.1:2222` → VM `10.0.2.15:22` 포트 전달을 구성했다. 첫 VM `devops-vm01`은 Ubuntu Server 24.04.5, Static IP `10.0.2.15/24`, SSH, 인터넷/DNS, Asia/Seoul/NTP, 시스템 업데이트 및 기본 관리 도구 설정까지 완료했다.

> ⏸️ **보류**
>
> Git / GitHub 설정, Notion 설정, VM의 프로젝트 내 역할 및 Docker/Kubernetes 등 추가 구성은 아키텍처 검토 후 진행한다.

## 🖥️ Ubuntu GUI 설치 및 세팅 과정

### 1. Ubuntu 설치 USB 제작

#### 설치 이미지

```text
Ubuntu 24.04.5.1 Desktop
```

#### 설치 USB

```text
SanDisk USB 64GB
```

#### Rufus 설정

```text
Rufus 4.15

Partition Scheme : GPT
Target System    : UEFI (비 CSM)
File System      : Large FAT32
```

USB로 부팅한 뒤 Ubuntu Desktop을 내부 NVMe SSD에 설치했다.

#### 설치 디스크 구성

```text
nvme0n1
├─ nvme0n1p1    1GB       /boot/efi
└─ nvme0n1p2    약 476GB  /
```

설치 완료 후 USB를 제거하고 내부 NVMe에서 정상적으로 부팅되는 것을 확인했다.

### 2. 기본 시스템 확인

Ubuntu 설치 후 시스템 정보를 확인했다.

```bash
hostnamectl
lsblk
ip addr
```

확인된 환경:

```text
OS           : Ubuntu 24.04.5 LTS
Architecture : x86-64
Disk         : NVMe 약 477GB
User         : devops
```

### 3. 시스템 업데이트

설치 직후 패키지를 최신 상태로 업데이트했다.

```bash
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
```

초기 업데이트 과정에서 Netplan, AppArmor, GNOME Shell 등의 시스템 패키지를 포함한 업데이트를 적용했다.

인터넷 연결 확인:

```bash
ping -c 4 8.8.8.8
```

```text
4 packets transmitted
4 received
0% packet loss
```

외부 네트워크 통신이 정상임을 확인했다.

### 4. Hostname 설정

초기 hostname에는 하드웨어 모델명이 포함되어 있었다.

프로젝트 서버임을 쉽게 식별할 수 있도록 다음과 같이 변경했다.

```bash
sudo hostnamectl set-hostname devops-server
```

`/etc/hosts`도 함께 수정했다.

```text
127.0.0.1 localhost
127.0.1.1 devops-server
```

확인:

```bash
hostname
getent hosts devops-server
```

최종 hostname:

```text
devops-server
```

### 5. 시간 및 한글 환경 설정

Timezone:

```text
Asia/Seoul (KST, +0900)
```

확인:

```bash
timedatectl
```

```text
System clock synchronized : yes
NTP service               : active
```

Ubuntu GUI에서 한글 입력 환경도 설정했다.

### 6. 물리 서버 네트워크 설정

#### 초기 DHCP 환경

Wi-Fi Interface:

```text
wlxb0386cf0123d
```

초기 DHCP IP:

```text
192.168.202.230/22
```

확인된 네트워크:

```text
Network : 192.168.200.0/22
Gateway : 192.168.200.1

DNS
├─ 164.124.101.2
└─ 8.8.8.8
```

NetworkManager 연결 프로필:

```text
rapa_classroom-5
```

#### Static IP 설정

물리 서버에서 사용할 IP를 다음과 같이 설정했다.

```text
IP      : 192.168.200.149/22
Gateway : 192.168.200.1
DNS     : 164.124.101.2
          8.8.8.8
```

```bash
sudo nmcli connection modify "rapa_classroom-5" \
  ipv4.method manual \
  ipv4.addresses 192.168.200.149/22 \
  ipv4.gateway 192.168.200.1 \
  ipv4.dns "164.124.101.2 8.8.8.8"
```

설정 후 Gateway → Internet → DNS 순서로 통신을 검증했다.

```bash
ping -c 4 192.168.200.1
ping -c 4 8.8.8.8
ping -c 4 google.com
```

모두 `0% packet loss` 확인.

### 7. OpenSSH 구성

외부 PC에서 물리 서버를 원격으로 관리할 수 있도록 OpenSSH Server를 구성했다.

```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh
```

확인:

```bash
systemctl is-enabled ssh
systemctl is-active ssh
```

```text
enabled
active
```

22번 포트:

```bash
sudo ss -tlnp | grep :22
```

외부 PC에서 실제 접속:

```bash
ssh devops@192.168.200.149
```

정상 접속 확인.

### 8. UFW 방화벽 설정

SSH 접근을 먼저 허용한 뒤 UFW를 활성화했다.

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

```text
External PC
     │
     │ TCP 22
     ▼
    UFW
     │
     │ OpenSSH ALLOW
     ▼
    sshd
```

UFW 활성화 이후 새로운 SSH 세션을 생성해 접속이 정상적으로 유지되는 것도 검증했다.

### 9. 기본 관리 도구 설치

```bash
sudo apt install -y \
curl wget vim git net-tools htop tree unzip zip jq
```

| 도구 | 용도 |
|---|---|
| curl | HTTP/API 요청 |
| wget | 파일 다운로드 |
| vim | 설정 파일 편집 |
| git | 형상 관리 |
| net-tools | 네트워크 확인 |
| htop | 시스템 자원/프로세스 확인 |
| tree | 디렉터리 구조 확인 |
| zip, unzip | 압축 |
| jq | JSON 처리 |

### 10. Docker 환경 구축

Docker 공식 Repository를 등록해 Docker Engine을 설치했다.

최종 환경:

```text
Docker Engine  : 29.8.1
Docker Compose : v5.5.1
containerd     : 2.3.6
```

서비스:

```bash
systemctl is-enabled docker
systemctl is-active docker
```

```text
enabled
active
```

`devops` 사용자가 `sudo` 없이 Docker를 사용할 수 있도록:

```bash
sudo usermod -aG docker $USER
```

설정했다.

최종 동작 검증:

```bash
docker run hello-world
```

```text
Hello from Docker!
```

정상 실행 확인.

### 11. GUI / 개발 도구 설치

#### Google Chrome

설치 완료.

#### IntelliJ IDEA

```text
IntelliJ IDEA 2026.2.3
```

USB에 준비한 Linux x86-64 패키지를 `/opt`에 설치했다.

GUI 실행 확인 완료.

#### Visual Studio Code

USB에 준비한:

```text
code_1.139.1-1790309529_amd64.deb
```

설치.

GUI 실행 확인 완료.

#### Postman

```bash
sudo snap install postman
```

GUI 실행 확인 완료.

#### VirtualBox

```text
VirtualBox 7.2.20
```

Ubuntu 24.04 Noble / amd64용 패키지를 사용했다.

확인:

```bash
VBoxManage --version
```

```text
7.2.20r175154
```

가상화:

```text
Intel VT-x
```

VirtualBox Kernel Module:

```text
vboxdrv
vboxnetflt
vboxnetadp
```

모두 정상 확인.

#### 보류

```text
Git / GitHub 계정 설정    ⏸️
Notion                   ⏸️
```

## 🧱 VM 설치 및 세팅 과정

### 1. Ubuntu Server ISO 준비

USB에 준비한:

```text
ubuntu-24.04.5-live-server-amd64.iso
```

를 사용했다.

VM 설치 중 USB 연결 문제를 방지하기 위해 ISO를 물리 서버 내부 SSD로 복사했다.

```bash
mkdir -p ~/iso

cp "/media/devops/Samsung USB/Ubuntu 24.04.5.1/ubuntu-24.04.5-live-server-amd64.iso" ~/iso/
```

### 2. VM 생성

초기 VM 사양:

```text
CPU  : 2 vCPU
RAM  : 4GB
Disk : 40GB
OS   : Ubuntu Server 24.04.5 LTS
```

Ubuntu Server 설치 후:

```text
User     : devops
Hostname : devops-vm01
```

로 구성했다.

### 3. VirtualBox NAT Network 구성

향후 여러 VM이 같은 가상 네트워크에서 통신할 수 있도록 일반 NAT가 아닌 **NAT Network**를 사용했다.

```text
Name    : UbuntuNetwork
Network : 10.0.2.0/24
Gateway : 10.0.2.1
```

VM:

```text
Adapter 1
└─ NAT Network
   └─ UbuntuNetwork
```

### 4. VM Static IP 설정

최종적으로 `devops-vm01`에 다음 주소를 할당했다.

```text
IP      : 10.0.2.15/24
Gateway : 10.0.2.1

DNS
├─ 8.8.8.8
└─ 1.1.1.1
```

Netplan:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 10.0.2.15/24
      routes:
        - to: default
          via: 10.0.2.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

적용:

```bash
sudo netplan generate
sudo netplan apply
```

확인:

```bash
ip -4 addr show enp0s3
ip route
```

```text
enp0s3
└─ 10.0.2.15/24

default via 10.0.2.1 dev enp0s3
```

인터넷 및 DNS 통신도 정상 확인했다.

### 5. VM OpenSSH 설정

VM에서 OpenSSH Server를 활성화했다.

확인:

```bash
systemctl is-enabled ssh
systemctl is-active ssh
```

SSH 22번 포트도 정상적으로 `LISTEN` 상태임을 확인했다.

### 6. NAT Network Port Forwarding

물리 서버에서 NAT Network 내부 VM으로 SSH 접속하기 위해 다음 규칙을 구성했다.

```text
Name       : ssh
Protocol   : TCP

Host
127.0.0.1:2222

       ↓

Guest
10.0.2.15:22
```

물리 서버에서:

```bash
ssh -p 2222 devops@127.0.0.1
```

실행.

정상적으로:

```text
devops@devops-vm01:~$
```

에 접속되는 것을 확인했다.

### 7. VM 기본 초기 설정

Timezone:

```bash
sudo timedatectl set-timezone Asia/Seoul
```

```text
Asia/Seoul (KST, +0900)
System clock synchronized : yes
NTP service               : active
```

OS 업데이트:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
```

기본 관리 도구:

```bash
sudo apt install -y \
curl wget vim git net-tools htop tree unzip zip jq
```

최종 확인:

```bash
hostname
ip -4 addr show enp0s3
ip route
systemctl is-active ssh
```

결과:

```text
Hostname : devops-vm01
IP       : 10.0.2.15/24
Gateway  : 10.0.2.1
SSH      : active
```

## 🔧 트러블슈팅

### 1. Static IP 설정 시 nmcli 오류

#### 문제

```bash
sudo nmcli connection modify "rapa_classroom-5" ipv4.method manual
```

실행 시:

```text
ipv4.method: method 'manual'
requires at least an address or a route
```

#### 원인

`manual` 방식으로 전환할 때 사용할 IPv4 주소 또는 Route가 함께 지정되어 있지 않았다.

#### 해결

Method와 네트워크 정보를 한 번에 설정했다.

```bash
sudo nmcli connection modify "rapa_classroom-5" \
  ipv4.method manual \
  ipv4.addresses 192.168.200.149/22 \
  ipv4.gateway 192.168.200.1 \
  ipv4.dns "164.124.101.2 8.8.8.8"
```

#### 결과

```text
192.168.200.149/22
```

Static IP 정상 적용.

### 2. Docker Permission Denied

#### 문제

```bash
docker ps
```

```text
permission denied while trying to connect
to the docker API at unix:///var/run/docker.sock
```

#### 원인

`devops` 사용자가 Docker socket에 접근할 수 있는 `docker` 그룹에 포함되어 있지 않았다.

#### 해결

```bash
sudo usermod -aG docker $USER
```

그룹 등록 후에도 기존 GUI 로그인 세션에서는 새로운 그룹 권한이 즉시 반영되지 않았다.

SSH 신규 세션에서는 `docker` 그룹이 정상적으로 반영되었으며, 물리 서버 재부팅 후 GUI 세션에서도 정상화됐다.

#### 검증

```bash
groups
docker ps
docker run hello-world
```

정상 실행 확인.

### 3. NAT Network VM에서 IPv4 미할당

#### 문제

VM 생성 후:

```bash
ip -4 addr
```

확인 시:

```text
127.0.0.1
```

만 존재하고 `enp0s3`에 IPv4 주소가 없었다.

따라서:

```bash
ip route
```

에도 Default Route가 존재하지 않았고:

```text
Network is unreachable
```

오류가 발생했다.

#### 확인 과정

NIC:

```bash
ip link
ls /sys/class/net
```

```text
lo
enp0s3
```

`enp0s3` 상태:

```text
UP
LOWER_UP
```

따라서 VirtualBox NIC와 Virtual Cable 자체는 정상.

Netplan:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
```

VirtualBox NAT Network도:

```text
UbuntuNetwork
10.0.2.0/24
Gateway: 10.0.2.1
```

로 존재했다.

#### 해결

DHCP 사용 대신 VM에 Static IP를 설정했다.

```text
10.0.2.15/24
```

Netplan 적용 후:

```bash
ip -4 addr show enp0s3
```

```text
inet 10.0.2.15/24
```

정상 확인.

### 4. Host → VM SSH 접속 실패

#### 문제

```bash
ssh -p 2222 devops@127.0.0.1
```

실행 시 연결되지 않았다.

#### 확인

VM SSH 서비스:

```bash
systemctl is-active ssh
```

```text
active
```

22번 포트 역시 정상 `LISTEN`.

즉 SSH Server 자체의 문제는 아니었다.

#### 원인

VM에 IPv4가 존재하지 않았기 때문에 Port Forwarding 목적지:

```text
10.0.2.15:22
```

가 실제로 존재하지 않았다.

#### 해결

VM Static IP `10.0.2.15/24`를 정상 적용한 뒤 다시 SSH 접속.

```bash
ssh -p 2222 devops@127.0.0.1
```

정상 접속 확인.

### 5. Physical Host와 VM 터미널 혼동

#### 문제

물리 서버와 VM에서 모두 `devops` 사용자를 사용하다 보니 명령을 잘못된 환경에서 실행하는 일이 발생했다.

#### 구분

```text
Physical Host
devops@devops-server:~$

VM
devops@devops-vm01:~$
```

앞으로 명령 실행 전 hostname을 확인해 작업 위치를 구분한다.

## 🗺️ 최종 구조

```text
                     ┌─────────────────────┐
                     │   강의실 Network    │
                     │ 192.168.200.0/22    │
                     └──────────┬──────────┘
                                │
                                │ Wi-Fi
                                ▼
┌─────────────────────────────────────────────────┐
│ Physical Host                                   │
│                                                 │
│ devops-server                                   │
│ Ubuntu Desktop 24.04.5                          │
│ 192.168.200.149/22                              │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ System                                      │ │
│ │ SSH / UFW / NTP / Basic Tools              │ │
│ └─────────────────────────────────────────────┘ │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ Development                                 │ │
│ │ IntelliJ / VS Code / Postman / Chrome      │ │
│ └─────────────────────────────────────────────┘ │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ Docker                                      │ │
│ │ Engine 29.8.1 / Compose / containerd       │ │
│ └─────────────────────────────────────────────┘ │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ VirtualBox 7.2.20                           │ │
│ │                                             │ │
│ │ UbuntuNetwork : 10.0.2.0/24                │ │
│ │ Gateway       : 10.0.2.1                   │ │
│ │                                             │ │
│ │        ┌───────────────────────────┐        │ │
│ │        │ devops-vm01              │        │ │
│ │        │ Ubuntu Server 24.04.5    │        │ │
│ │        │ 10.0.2.15/24             │        │ │
│ │        │ SSH :22                  │        │ │
│ │        └───────────────────────────┘        │ │
│ │                                             │ │
│ │ 127.0.0.1:2222 ───────▶ 10.0.2.15:22      │ │
│ └─────────────────────────────────────────────┘ │
│                                                 │
└─────────────────────────────────────────────────┘
```

> ✅ **현재 단계**
>
> 물리 서버 및 첫 번째 Ubuntu Server VM의 Base Infrastructure 구축 완료

> ➡️ **다음 단계**
>
> All Architecture를 실제 구축 환경과 매핑하여 VM 구성 및 각 기술의 배치 위치 결정
