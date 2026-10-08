# DiskAlmostFull

> 담당: 손지원 · 기준일: 2026-10-08 · 원본: [노션 「모니터링 경보 대응 Runbook」](https://app.notion.com/p/3f33f39c862e8087a141ee4393080dd8)

| 경보 | 등급 | 조건 |
| --- | --- | --- |
| DiskAlmostFull | Warning | 모든 머신, 남은 공간 15% 미만이 10분 지속 |
| DiskAlmostFull | Critical | db-01·k8s-worker1~3, 남은 공간 10% 미만이 5분 지속 |

- 같은 디스크에 Critical이 울리면 Warning은 Slack에 보내지 않음.

## 증상

- Slack에 `[FIRING] DiskAlmostFull`과 머신 이름이 옴. 어느 디스크인지는 Prometheus에서 `mountpoint` 라벨로 봄.

## 영향

| 머신 | 꽉 차면 |
| --- | --- |
| db-01 | MySQL이 쓰기를 멈춤 → 앱 오류 |
| k8s-worker | kubelet이 Pod를 내보냄(퇴출), 이미지를 못 받음 |
| mon-01 | Prometheus·Loki 저장 실패. Prometheus는 25GB 상한이 있어 보통 먼저 지움 |
| ci-01 | 빌드 실패 |

## 확인

```bash
# 1. 디스크 사용량 (대상 머신)
ssh devops@<IP> 'df -h -x tmpfs -x overlay -x squashfs'

# 2. 큰 폴더 찾기
ssh devops@<IP> 'sudo du -xh --max-depth=2 / 2>/dev/null | sort -h | tail -15'

# 3. 머신별로 자주 차는 곳
#   worker : sudo crictl images / sudo du -sh /var/lib/containerd /var/log/pods
#   db-01  : sudo du -sh /var/lib/mysql /var/log/mysql
#   mon-01 : sudo du -sh /opt/monitoring/*   그리고 sudo docker system df
#   공통   : sudo journalctl --disk-usage
```

- Prometheus에서 `node_filesystem_avail_bytes{instance="<머신 이름>"} / node_filesystem_size_bytes{instance="<머신 이름>"}`로 얼마나 빨리 줄었는지 봄. 갑자기 줄었으면 로그 폭주·백업 파일을 의심함.

## 조치

| 원인 | 조치 |
| --- | --- |
| 시스템 로그 | `sudo journalctl --vacuum-size=200M` |
| worker의 안 쓰는 이미지 | `sudo crictl rmi --prune` |
| mon-01 Docker 찌꺼기 | `sudo docker system prune` (실행 중 컨테이너·볼륨은 지우지 않음) |
| db-01 백업 파일 | 보관 정책(일일 7개·주간 4개)보다 많은지 확인. 백업 담당에게 알림 |
| 계속 부족 | 디스크 확장은 VM 담당과 협의 |

- **지우기 전에 무엇인지 확인함.** `/var/lib/mysql`, `/opt/monitoring`의 데이터 폴더, etcd(`/var/lib/etcd`)는 지우지 않음.

## 복구 확인

- `df -h`에서 사용률 85% 미만.
- Slack에 `[RESOLVED] DiskAlmostFull`이 옴.
