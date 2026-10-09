# MySQLDown

> 담당: 손지원 · 기준일: 2026-10-08 · 원본: [노션 「모니터링 경보 대응 Runbook」](https://app.notion.com/p/3f33f39c862e8087a141ee4393080dd8)

| 경보 | 등급 | 조건 |
| --- | --- | --- |
| MySQLDown | Critical (`#alert-critical`) | 1분 동안 `mysql_up == 0`(exporter는 살았는데 DB 응답 없음) 또는 `up{job="mysqld"} == 0`(exporter 응답 없음) |

## 증상

- `#alert-critical`에 `[FIRING] MySQLDown`이 옴.
- db-01 머신 자체가 꺼졌으면 [instance-down](instance-down.md)도 함께 옴.

## 영향

- `mysql_up == 0`이면 앱이 DB를 못 씀 → prod 오류. **바로 대응함.**
- `up{job="mysqld"} == 0`만이면 DB는 살아 있고 감시만 끊겼을 수 있음.

## 확인

```bash
# 1. 어느 쪽 문제인지 (Prometheus 화면)
#    mysql_up                → 0 이면 DB 문제
#    up{job="mysqld"}        → 0 이면 exporter 또는 네트워크 문제

# 2. db-01 에서 MySQL
ssh devops@192.168.56.31 'systemctl status mysql --no-pager'
ssh devops@192.168.56.31 'sudo mysql -e "SELECT 1"'
ssh devops@192.168.56.31 'sudo tail -50 /var/log/mysql/error.log'

# 3. exporter
ssh devops@192.168.56.31 'systemctl status mysqld_exporter --no-pager'
curl -s http://192.168.56.31:9104/metrics | grep mysql_up        # mon-01 에서
```

## 조치

| 원인 | 조치 |
| --- | --- |
| MySQL 멈춤 | `ssh devops@192.168.56.31 'sudo systemctl restart mysql'` 후 error.log 확인 |
| 디스크 꽉 참 | [disk-almost-full](disk-almost-full.md) 먼저 |
| exporter만 멈춤 | `ssh devops@192.168.56.31 'sudo systemctl restart mysqld_exporter'` |
| mon-01에서 9104 접속 안 됨 | db-01 ufw에 `192.168.56.41 → 9104` 허용 확인 (`sudo ufw status`). 없으면 MySQL role 담당에게 알림 |
| 데이터 손상 | 직접 고치지 않음. 백업 복구 절차(MySQL 백업 담당)를 따름 |

## 복구 확인

- `mysql_up`이 1, `up{job="mysqld"}`가 1.
- 앱 health(`/actuator/health`)가 UP.
- Slack에 `[RESOLVED] MySQLDown`이 옴.
