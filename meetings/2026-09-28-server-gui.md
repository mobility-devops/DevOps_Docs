# 9월 28일 Server GUI 작업 정리

> 원본: 노션 「회의 기록 / 09-28 / 9월 28일 Server GUI 작업 정리」에서 2026-09-30 기준으로 옮김.

> 9월 28일 프로젝트 대화 기록에서 **실제로 실행하거나 결과를 확인한 작업**을 기준으로 정리

### 1. Ubuntu GUI 설치 및 기본 설정

#### 운영체제 설치

- 프로젝트 서버에 `Ubuntu 24.04.5 LTS Desktop` 설치
- 내부 NVMe 디스크에 Ubuntu 설치
- 설치 USB를 제거한 뒤 정상 재부팅 확인
- 사용자 계정: `devops`
- 기존 호스트명: `devops-B80LV-AR45B5E`
- 최종 호스트명: `devops-server`

#### 시스템 업데이트

```text
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
```

- 처음에 `sudo apt upgrade-y`로 입력하여 다음 오류 발생

```text
E: Invalid operation upgrade-y
```

- `sudo apt upgrade -y`로 수정하여 정상 진행
- 최종적으로 21개 패키지 업데이트 완료

#### 시간 설정

- 시간대: `Asia/Seoul`
- 시스템 시간 동기화: 정상
- NTP 서비스: `active`

```text
Time zone: Asia/Seoul (KST, +0900)
System clock synchronized: yes
NTP service: active
```

#### 한글 입력 설정

- Ubuntu GUI에서 한국어 입력기 설정 완료
- 한글/영문 입력 전환이 가능한 상태까지 확인

---

### 2. 서버 하드웨어 및 저장공간 확인

서버 사양을 확인한 결과는 다음과 같다.

| 항목 | 확인 결과 |
|---|---|
| CPU | Intel Core i5-14400 |
| 논리 CPU | 16개 |
| 메모리 | 약 62GiB |
| 사용 가능 메모리 | 약 58GiB |
| SSD | 약 468GB |
| 사용 가능 공간 | 약 422GB |

USB에는 다음 Ubuntu 이미지가 있었다.

- `ubuntu-24.04.5-live-server-amd64.iso` 약 3.9GB
- Ubuntu Desktop ISO 약 5.9GB

VirtualBox에서 사용하기 위해 Ubuntu Server ISO를 서버 내부 저장공간으로 복사해 사용했다.

---

### 3. 네트워크 고정 IP 설정

#### Wi-Fi 연결

- 사용한 Wi-Fi 프로필: `rapa_classroom-5`
- 네트워크 인터페이스: `wlxb0386cf0123d`

#### 고정 네트워크 정보

| 구분 | 설정값 |
|---|---|
| 서버 IP | `192.168.200.149/22` |
| Gateway | `192.168.200.1` |
| DNS 1 | `164.124.101.2` |
| DNS 2 | `8.8.8.8` |

#### 연결 검증

다음 대상을 대상으로 통신을 확인했다.

```text
ping 192.168.200.1
ping 8.8.8.8
ping google.com
```

- Gateway 통신 정상
- 외부 인터넷 통신 정상
- DNS 이름 해석 정상
- 모두 패킷 손실 `0%`

---

### 4. OpenSSH 원격 접속 구성

#### OpenSSH 설치 및 서비스 확인

- OpenSSH Server 설치
- 부팅 시 자동 실행 설정
- SSH 서비스가 `active` 상태인지 확인
- IPv4와 IPv6 모두 22번 포트에서 대기하는 것을 확인

```text
enabled
active
0.0.0.0:22 LISTEN
[::]:22 LISTEN
```

#### 외부 PC 접속 확인

다른 PC에서 다음과 같이 접속했다.

```text
ssh devops@192.168.200.149
```

- 외부 PC에서 서버로 SSH 접속 성공
- 서버가 재부팅돼도 SSH 서비스가 자동 실행되도록 설정 완료

---

### 5. UFW 방화벽 설정

처음 SSH 상태를 점검했을 때 UFW는 `inactive` 상태였다. 이후 물리 호스트 `devops-server`에서 UFW를 활성화했다.

#### 최종 확인된 허용 규칙

- OpenSSH 허용
- 교실 내부 네트워크에서 VirtualBox SSH 포트 접근 허용

```text
OpenSSH                   ALLOW       Anywhere
1111/tcp                  ALLOW       192.168.200.0/22
2222/tcp                  ALLOW       192.168.200.0/22
3333/tcp                  ALLOW       192.168.200.0/22
OpenSSH (v6)              ALLOW       Anywhere (v6)
```

#### 작업 중 발생한 실수

처음에는 `1111`, `2222`, `3333` 허용 명령을 `controlnode` VM 내부에서 실행했다.

- `controlnode`의 UFW는 `inactive`
- 포트포워딩을 담당하는 위치는 VM이 아니라 VirtualBox가 설치된 물리 호스트
- 따라서 물리 호스트 `devops-server`에서 다시 방화벽 규칙을 설정

Tailscale만 사용한다면 `1111~3333` 포트를 LAN에 열 필요는 없지만, 교실 내부 네트워크에서 VM으로 직접 접속하기 위해 해당 규칙을 유지했다.

---

### 6. 기본 관리 도구 설치

서버 관리와 이후 설치 작업에 사용할 기본 도구를 준비했다.

```text
sudo apt install -y \
  curl wget vim git net-tools htop tree unzip zip jq
```

다만 여기서 설치한 `git` 실행 파일과 별개로, 다음 설정은 진행하지 않았다.

- Git 사용자 이름·이메일 설정
- GitHub 계정 연동
- SSH Key 생성 및 등록
- 프로젝트 저장소 연결

즉, **기본 도구 설치는 완료됐지만 Git/GitHub 계정 설정은 보류**한 상태다.

---

### 7. GUI 프로그램 설치

| 프로그램 | 상태 | 확인 내용 |
|---|---|---|
| Google Chrome | ✅ | 설치 완료 |
| IntelliJ IDEA | ✅ | `2026.2.3` 실행 확인 |
| Visual Studio Code | ✅ | 설치 완료 |
| VirtualBox | ✅ | `7.2.20` 설치 및 실행 확인 |
| Postman | ✅ | GUI 실행 확인 |
| Notion | ⏸️ | 공식 Linux 앱이 없어 설치 보류 |

#### Notion 확인 결과

Ubuntu App Center에 표시되는 Notion 관련 앱도 확인했지만, 공식 Linux용 Notion 데스크톱 앱으로 보기 어려웠다.

따라서 서버에는 설치하지 않고 다음으로 미뤘다.

- 웹 브라우저에서 Notion 사용
- Linux 공식 지원 여부를 확인한 뒤 설치
- 필요하면 별도의 비공식 래퍼 사용 여부 검토

---

### 8. Docker 설치

#### 설치 및 버전 확인

| 구성요소 | 버전 |
|---|---|
| Docker Engine | `29.8.1` |
| Docker Compose | `v5.5.1` |
| containerd | `v2.3.6` |

- Docker 공식 Ubuntu `noble` 저장소 등록
- Docker 서비스 활성화
- 부팅 시 자동 실행 설정

```text
Docker daemon: active
Docker daemon: enabled
```

#### 사용자 권한 설정

`devops` 사용자를 `docker` 그룹에 추가했다.

```text
sudo usermod -aG docker devops
```

이후 새 세션에서 다음 명령이 `sudo` 없이 동작하는 것을 확인했다.

```text
docker ps
```

테스트 컨테이너도 실행했다.

```text
docker run hello-world
```

따라서 물리 호스트의 Docker 설치와 일반 사용자 권한 설정까지 완료된 상태다.

---

### 9. VirtualBox 설치 및 검증

#### VirtualBox 상태

- 버전: `7.2.20`
- Ubuntu용 `.deb` 패키지 사용
- VirtualBox GUI 정상 실행
- CPU 가상화 기능 VT-x 사용 가능
- VirtualBox 커널 모듈 정상 로드 확인

#### 첫 번째 VM 구성 과정

처음에는 Ubuntu Server VM을 한 대 생성했다.

- 초기 이름: `ubuntu-server-01`
- 이후 작업 중 이름: `devops-vm01`
- OS: Ubuntu Server 24.04.5 LTS
- 사용자: `devops`

초기 VM에서 SSH 서비스는 정상 동작했지만 네트워크 인터페이스에 IPv4가 할당되지 않아 다음 접속이 실패했다.

```text
ssh -p 2222 devops@127.0.0.1
```

당시 확인 결과:

- VM의 SSH 서비스: `active`
- VM의 22번 포트: `LISTEN`
- `ip -4 addr show` 결과에는 `lo`만 존재
- `enp0s3`에 IPv4가 없는 상태

원인은 DHCP를 사용하지 않는 `NAT Network`에 VM을 연결했지만, VM 내부 정적 IP 설정이 아직 적용되지 않았기 때문이었다.

---

### 10. VirtualBox VM 3대 구성

최종적으로 다음 3대의 Ubuntu VM을 구성했다.

| VM | 내부 IP | Prefix | Gateway | 호스트 SSH 포트 |
|---|---|---|---|---|
| `controlnode` | `10.0.2.15` | `/24` | `10.0.2.1` | `1111` |
| `managednode1` | `10.0.2.16` | `/24` | `10.0.2.1` | `2222` |
| `managednode2` | `10.0.2.17` | `/24` | `10.0.2.1` | `3333` |

#### VM 네트워크 구조

- VirtualBox 네트워크 유형: `NAT Network`
- 세 VM을 동일 NAT Network에 연결
- DHCP 대신 Netplan 정적 IP 사용
- 각 VM의 hostname과 IP 설정 완료
- SSH 재접속 확인

포트포워딩 구조는 다음과 같다.

```text
devops-server:1111 → controlnode:22
devops-server:2222 → managednode1:22
devops-server:3333 → managednode2:22
```

호스트 내부에서는 다음과 같이 접속할 수 있다.

```text
ssh -p 1111 devops@127.0.0.1
ssh -p 2222 devops@127.0.0.1
ssh -p 3333 devops@127.0.0.1
```

#### 최종 VM 자원 상태

세 VM에서 확인한 공통 사양은 다음과 같다.

| 항목 | VM당 할당량 |
|---|---|
| vCPU | 4개 |
| 메모리 | 약 3.8GiB |
| 디스크 | 25GB |
| 디스크 여유 공간 | 약 17GB |

#### 아직 설치하지 않은 구성요소

세 VM에서는 다음 항목이 아직 설치되지 않은 상태였다.

- Docker
- containerd
- kubeadm
- kubectl
- kubelet
- Kubernetes 클러스터

추가로 각 VM에는 약 3.8GiB의 Swap이 활성화되어 있었다. Kubernetes를 설치하려면 이후 Swap 비활성화가 필요하다.

---

### 11. Tailscale VPN 구성

외부 포트포워딩 없이 서버에 안전하게 접속하기 위해 Tailscale을 설치했다.

#### 설치 위치

- VM이 아닌 물리 호스트 `devops-server`에만 설치
- 기존 VM 네트워크에는 영향을 주지 않도록 설정

#### 연결 옵션

```text
sudo tailscale up \
  --accept-dns=false \
  --accept-routes=false
```

- Tailscale DNS를 서버에 강제로 적용하지 않음
- 다른 Tailscale 장비의 라우팅 경로를 받아오지 않음
- 서버의 기본 Gateway `192.168.200.1` 유지

#### 연결 정보

| 항목 | 값 |
|---|---|
| Tailscale IP | `100.120.189.82` |
| MagicDNS 장치명 | `devops-server` |
| Windows 노트북 Tailscale IP | `100.95.45.39` |

외부에서는 다음 명령으로 서버에 접속할 수 있도록 구성했다.

```text
ssh devops@100.120.189.82
```

접속 시험 과정에서는 네트워크에 따라 직접 LAN 접속이 실패하기도 했지만, `rapa_meetingroom-3` 환경으로 변경한 뒤 SSH 접속 성공을 확인했다.

Tailscale은 GUI 창이 일반 앱처럼 열리지 않을 수 있지만, 시스템 서비스로 실행되며 부팅 후에도 자동 시작되는 상태다.

---

### 12. 9월 28일 최종 상태

```text
Ubuntu 24.04.5 LTS — devops-server
│
├─ 기본 OS
│  ├─ Ubuntu Desktop GUI 설치          ✅
│  ├─ 시스템 업데이트                  ✅
│  ├─ hostname: devops-server         ✅
│  ├─ Asia/Seoul + NTP                ✅
│  └─ 한글 입력                        ✅
│
├─ Network
│  ├─ Wi-Fi 연결                       ✅
│  ├─ Static IP: 192.168.200.149/22   ✅
│  ├─ Gateway: 192.168.200.1          ✅
│  └─ DNS / Internet                  ✅
│
├─ Remote / Security
│  ├─ OpenSSH Server                  ✅
│  ├─ SSH 자동 시작                    ✅
│  ├─ UFW                             ✅
│  ├─ 외부 PC → SSH                   ✅
│  └─ Tailscale VPN                   ✅
│
├─ 기본 프로그램
│  ├─ Chrome                          ✅
│  ├─ IntelliJ IDEA                   ✅
│  ├─ VS Code                         ✅
│  ├─ Postman                         ✅
│  └─ 기본 관리 도구                   ✅
│
├─ Docker
│  ├─ Docker Engine 29.8.1            ✅
│  ├─ Docker Compose v5.5.1           ✅
│  ├─ containerd v2.3.6               ✅
│  └─ devops 사용자 권한               ✅
│
├─ VirtualBox 7.2.20                  ✅
│  ├─ controlnode   10.0.2.15:22      ✅
│  ├─ managednode1  10.0.2.16:22      ✅
│  └─ managednode2  10.0.2.17:22      ✅
│
├─ Git / GitHub 설정                  ⏸️ 보류
├─ Notion Desktop                     ⏸️ 보류
└─ VM 내부 Kubernetes 구성             ⏳ 미진행
```

핵심적으로 이날은 **물리 서버의 Ubuntu GUI 초기 구축부터 고정 IP·SSH·Docker·VirtualBox·VM 3대·Tailscale 원격 접속 기반까지 준비한 날**이다. 아직 Jenkins, Kubernetes, Argo CD 같은 실제 DevOps 플랫폼 설치 단계에는 들어가지 않았다.
