# 현재 환경 현황 (네트워크 · VM)

- 기준일: 2026-09-30
- 측정 방법: 호스트와 VM 3대에서 직접 실행한 명령 결과
- 범위: 현재 상태만 기록한다. 계획, 결정 사항, 개선 항목은 포함하지 않는다.
- 표기: "추정"으로 적은 항목은 명령 결과에서 직접 확인하지 못한 내용이다.

## 1. 한눈에 보기

| 구분 | 요약 |
|---|---|
| 호스트 | Ubuntu 24.04.5, i5-14400(10코어/16스레드), RAM 62GiB, NVMe SSD 476GB(395GB 여유) |
| 가상화 | VirtualBox 7.2.20, VT-x 켜짐 |
| VM | 3대 실행 중, 각 4vCPU / 4GB (Ansible 실습 환경으로 추정) |
| 네트워크 | 호스트 Wi-Fi + Host-Only(192.168.56.0/24) + NAT Network(10.0.2.0/24) + Tailscale |
| 여유 자원 | RAM 가용 약 50GiB, 디스크 395GB |

## 2. 구조

```
                     인터넷
                        │
              공유기/학교망 (192.168.200.1)
                        │ Wi-Fi
 ┌──────────────────────┴───────────────────────────────────┐
 │ 호스트 PC "devops-server" (Ubuntu 24.04)                   │
 │   Wi-Fi     192.168.200.149/22                            │
 │   Tailscale 100.120.189.82                                │
 │   vboxnet0  192.168.56.1   (Host-Only)                    │
 │   포트 1111/2222/3333 ─ NAT Network 포트포워딩 (SSH)         │
 │                                                          │
 │  ┌────────────── VirtualBox ─────────────────┐            │
 │  │ controlnode   10.0.2.15 / 192.168.56.101  │            │
 │  │ managednode1  10.0.2.16 / 192.168.56.102  │            │
 │  │ managednode2  10.0.2.17 / 192.168.56.103  │            │
 │  └───────────────────────────────────────────┘            │
 └──────────────────────────────────────────────────────────┘

 Windows PC (100.95.45.39) ─Tailscale─▶ 호스트  (측정 시점 offline)
```

## 3. 호스트 PC

### 3-1. 하드웨어와 OS

| 항목 | 값 |
|---|---|
| 호스트명 | devops-server |
| OS / 커널 | Ubuntu 24.04.5 LTS / 7.0.0-34-generic |
| CPU | Intel Core i5-14400 (10코어 / 16스레드) |
| RAM | 총 62GiB, 사용 11GiB, 가용 50GiB, Swap 8GiB |
| 디스크 | NVMe SSD, `/` 468GB 중 49GB 사용, 395GB 여유 |
| 시간대 | Asia/Seoul (KST), NTP 동기화됨 |
| 실행 중인 주요 프로그램 | IntelliJ IDEA, MySQL(호스트 로컬), Java 앱(8080), Docker |

### 3-2. 네트워크 인터페이스

| 인터페이스 | 주소 | 상태 | 역할 |
|---|---|---|---|
| Wi-Fi (`wlx…`, USB 어댑터) | 192.168.200.149/22 | UP | 인터넷 경로, 게이트웨이 192.168.200.1 |
| `eno1` | 없음 | DOWN | 유선 랜(미사용) |
| `tailscale0` | 100.120.189.82 | UP | Tailscale VPN |
| `vboxnet0` | 192.168.56.1/24 | UP | VM과 통신하는 Host-Only |
| `docker0` | 172.17.0.1/16 | DOWN | Docker 기본 브리지 |
| DNS | 164.124.101.2, 8.8.8.8 | | |

- 기본 경로: `192.168.200.1` (Wi-Fi 경유)
- IP 포워딩: `net.ipv4.ip_forward = 1`

### 3-3. 열려 있는 포트

| 포트 | 프로세스 | 노출 범위 | 비고 |
|---|---|---|---|
| 22 | sshd | 모든 인터페이스 | ufw에서 전체 허용 |
| 1111 / 2222 / 3333 | VBoxNetNAT | 모든 인터페이스 | VM SSH 포트포워딩, ufw는 192.168.200.0/22만 허용 |
| 3306, 33060 | mysqld | localhost만 | 호스트 로컬 MySQL |
| 8080 | java (`~/.jdks/temurin-21`) | 모든 인터페이스 | IDE 관리 JDK로 실행 중인 Java 앱. Spring Boot 개발 실행으로 추정. ufw가 외부 접근 차단 |
| 63342, 30000, 44417 등 | IntelliJ | localhost만 | IDE 내부 |
| 631 | cupsd | localhost만 | 프린터 서비스 |

### 3-4. 방화벽(ufw)

| # | 규칙 | 허용 대상 |
|---|---|---|
| 1, 5 | OpenSSH | 어디서든(IPv4/IPv6) |
| 2~4 | 1111, 2222, 3333/tcp | 192.168.200.0/22 |

- 상태: 활성
- 기본 정책: incoming 차단, outgoing 허용, routed 차단

## 4. VirtualBox 네트워크

| 이름 | 종류 | 대역 | 비고 |
|---|---|---|---|
| `vboxnet0` | Host-Only | 192.168.56.0/24 | 호스트 `.1`, DHCP 꺼짐 |
| `UbuntuNetwork` | NAT Network | 10.0.2.0/24 | 게이트웨이 10.0.2.1, DHCP 켜짐 |

`UbuntuNetwork` 포트 포워딩:

| 호스트 포트 | 대상 VM | 대상 포트 |
|---|---|---|
| 1111 | controlnode (10.0.2.15) | 22 |
| 2222 | managednode1 (10.0.2.16) | 22 |
| 3333 | managednode2 (10.0.2.17) | 22 |

## 5. VM

### 5-1. 요약

| VirtualBox 이름 | 호스트명 | NAT Network (`enp0s3`) | Host-Only (`enp0s8`) | 스펙 | 상태 |
|---|---|---|---|---|---|
| ubuntu-server-01 | controlnode | 10.0.2.15 | 192.168.56.101 | 4vCPU / 4GB | 실행 중 |
| managed-node-1 | managednode1 | 10.0.2.16 | 192.168.56.102 | 4vCPU / 4GB | 실행 중 |
| managed-node-2 | managednode2 | 10.0.2.17 | 192.168.56.103 | 4vCPU / 4GB | 실행 중 |

- 어댑터1은 NAT Network `UbuntuNetwork`, 어댑터2는 Host-Only `vboxnet0`이다.
- 두 대역 모두 고정 주소로 설정되어 있고(기본 경로 `proto static`), 기본 경로는 `10.0.2.1`이다.

### 5-2. VM 내부 (3대 모두 수집, 동일한 구성)

| 항목 | 값 |
|---|---|
| OS / 커널 | Ubuntu 24.04.5 LTS / Linux 6.8.0-142-generic |
| 자원 | 4 vCPU, RAM 3.8GiB(사용 약 450MiB), Swap 3.8GiB 켜짐 |
| 디스크 | 25GB 중 6.8GB 사용(17GB 여유) |
| DNS | 8.8.8.8, 1.1.1.1 (고정) |
| 시간대 | Asia/Seoul, 동기화됨 |
| 설치된 도구 | Python 3.12.3 (Ansible, Docker 없음) |
| 열린 포트 | 22(SSH)만 |

### 5-3. VM 디스크 사용량 (호스트 기준)

| 폴더 | 실제 사용량 | 비고 |
|---|---|---|
| managed-node-1 | 4.9GB | |
| managed-node-2 | 5.1GB | |
| ubuntu-server-01 | 4.9GB | |
| ubuntu-server-02 | 4.4GB | VirtualBox에 등록되지 않음. `.vbox`, `.vdi` 파일만 있고 마지막 수정은 2026-09-28 14:00 |
| 합계 | 약 19.3GB | |

- `vboxmanage list vms`로 확인한 등록 VM은 3대이다.

## 6. Tailscale

| 기기 | 주소 | OS | 상태 |
|---|---|---|---|
| devops-server | 100.120.189.82 | Linux | 온라인 |
| desktop-88jnjgo | 100.95.45.39 | Windows | offline (측정 시점) |

- VM은 tailnet에 등록되어 있지 않다.

## 7. 현재 접속 경로 (추정)

| 출발 | 경로 | 대상 |
|---|---|---|
| Windows PC | Tailscale → 호스트(100.120.189.82) | 호스트 |
| 호스트 | Host-Only `192.168.56.101~103` 또는 포트포워딩 `localhost:1111/2222/3333` | VM |
| 같은 LAN의 기기 | 호스트 `192.168.200.149`의 1111/2222/3333 (ufw가 192.168.200.0/22만 허용) | VM SSH |
