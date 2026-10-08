# WatchdogMissing (모니터링 자체 장애)

> 담당: 손지원 · 기준일: 2026-10-08 · 원본: [노션 「모니터링 경보 대응 Runbook」](https://app.notion.com/p/3f33f39c862e8087a141ee4393080dd8)

| 경보 | 보내는 곳 | 조건 |
| --- | --- | --- |
| `alertmanager-watchdog` is DOWN | healthchecks.io → `#alert-critical` | Alertmanager가 1분마다 보내는 생존 신호(Watchdog)가 6분(1분 + 유예 5분) 동안 오지 않음 |

- 이 경보는 Prometheus가 아니라 **바깥의 healthchecks.io**가 보냄. 그래서 Slack 메시지에 Runbook 링크가 없음.

## 증상

- `#alert-critical`에 healthchecks.io 앱이 보낸 `"alertmanager-watchdog" is DOWN`이 옴.

## 영향

- **다른 모든 경보가 오지 않는 상태임.** 장애가 나도 Slack으로 알 수 없음.
- 원인은 셋 중 하나임: Prometheus 멈춤, Alertmanager 멈춤, mon-01·호스트 PC 꺼짐(또는 인터넷 끊김).

## 확인

```bash
# 1. 호스트 PC 가 살아 있는지 (Tailscale 접속 PC에서)
ping -c 3 192.168.56.1

# 2. mon-01
ping -c 3 192.168.56.41
ssh devops@192.168.56.41 'sudo docker ps --format "{{.Names}}\t{{.Status}}"'
#    prometheus, alertmanager, loki, grafana, blackbox 5개가 Up 이어야 함

# 3. 멈춘 컨테이너 로그
ssh devops@192.168.56.41 'sudo docker logs --tail 50 alertmanager'
ssh devops@192.168.56.41 'sudo docker logs --tail 50 prometheus'

# 4. mon-01 에서 인터넷 (healthchecks.io 로 나가는 길)
ssh devops@192.168.56.41 'curl -sI -m 5 https://hc-ping.com | head -1'
```

## 조치

| 원인 | 조치 |
| --- | --- |
| 호스트 PC 꺼짐 | 호스트 PC를 켬. VM은 자동으로 켜짐. 다 켜지는 데 몇 분 걸림 |
| mon-01 VM 꺼짐 | 호스트 PC에서 `cd ~/DevOps_Infra && vagrant up mon-01` |
| 컨테이너 멈춤 | `ssh devops@192.168.56.41 'cd /opt/monitoring && sudo docker compose up -d'` |
| 설정 오류로 재시작 반복 | 최근 DevOps_Infra monitoring role 변경을 되돌리고 플레이북 재실행 |
| 인터넷 끊김 | 호스트 PC의 외부 연결 확인. 내부 경보는 정상일 수 있음 |

## 복구 확인

- healthchecks.io에서 `alertmanager-watchdog`가 **up(초록)**.
- Slack에 `"alertmanager-watchdog" is UP`이 옴.
- 복구 전 시간 동안 놓친 경보가 없는지 Prometheus `http://192.168.56.41:9090/alerts`에서 확인.

## 다른 healthchecks.io 경보

| 체크 | 뜻 | 대응 | 신호 담당 |
| --- | --- | --- | --- |
| `host-heartbeat` | 호스트 PC cron 신호 끊김 = 호스트 PC 꺼짐 또는 인터넷 끊김 | 이 문서의 확인 1번, 조치의 「호스트 PC 꺼짐」·「인터넷 끊김」 | Ansible 담당 |
| `backup-db` | DB 백업 신호 없음 = 백업 실패 | 백업 담당의 백업·복구 절차 | Ansible 담당 (DevOps_Infra #26) |
| `backup-keys` | Sealed Secrets 열쇠 백업 신호 없음 | GitOps `secrets/README.md`의 열쇠 백업 절차 | CD·Ansible 협의 |
