# InstanceDown

> 담당: 손지원 · 기준일: 2026-10-09 · 원본: [노션 「모니터링 경보 대응 Runbook」](https://app.notion.com/p/3f33f39c862e8087a141ee4393080dd8)

| 경보 | 등급 | 조건 |
| --- | --- | --- |
| InstanceDown | Warning (`#alert-warning`) | 호스트 쪽 머신(`zone="host"`)의 node_exporter가 1분 동안 응답 없음: `up{job="node", zone="host"} == 0` |

## 증상

- Slack `#alert-warning`에 `[FIRING] InstanceDown`과 머신 이름(`instance`)이 옴.
- 대상: devops-server, ci-01, k8s-master, k8s-worker1·2, db-01, mon-01. 노트북 쪽(laptop-01, k8s-worker3)은 [laptop-down](laptop-down.md)에서 다룸.

## 영향

| 머신 | 영향 |
| --- | --- |
| ci-01 | 빌드·배포 파이프라인 멈춤. 서비스는 계속됨 |
| k8s-master | 새 배포·스케줄링 멈춤. 이미 떠 있는 Pod는 계속 응답함 |
| k8s-worker1·2 | 그 노드의 Pod가 다른 worker로 옮겨 감. 1대면 서비스 유지 |
| db-01 | 앱 DB 접속 실패 → [mysql-down](mysql-down.md)도 함께 옴 |
| mon-01 | 지표 수집·경보 중단. 몇 분 뒤 healthchecks.io가 [watchdog-missing](watchdog-missing.md)을 보냄 |

- node_exporter만 죽고 머신은 살아 있을 수도 있음. 먼저 머신이 살아 있는지 확인함.

## 확인

```bash
# 1. 머신 응답 (호스트 PC 또는 Tailscale 접속 PC에서. devops-server 에서는 ssh 에 -i ~/.ssh/id_devops 를 붙임)
ping -c 3 <IP>
ssh devops@<IP> 'uptime'

# 2. VM 상태 (호스트 PC, devops-server 의 ~/DevOps_Infra 에서만. worktree 폴더에서는 vagrant 금지)
cd ~/DevOps_Infra && vagrant status <머신 이름>

# 3. 머신이 살아 있으면 node_exporter 확인
ssh devops@<IP> 'systemctl status prometheus-node-exporter --no-pager'
ssh devops@<IP> 'curl -s localhost:9100/metrics | head -3'

# 4. node_exporter 가 살아 있으면 mon-01 에서 닿는지 확인 (000 이면 대상의 방화벽)
ssh devops@192.168.56.41 'curl -s -m 3 -o /dev/null -w "%{http_code}\n" http://<IP>:9100/metrics'
```

- Prometheus 화면(`http://192.168.56.41:9090/targets`)에서 job `node`의 해당 대상 오류 메시지를 봄.
- ufw가 켜진 머신은 devops-server(수동), ci-01·db-01·mon-01(role)뿐임. k8s 노드와 노트북은 ufw를 켜지 않음.

## 조치

| 원인 | 조치 |
| --- | --- |
| VM이 꺼짐 (`poweroff`) | `cd ~/DevOps_Infra && vagrant up <머신 이름>` |
| VM이 멈춤 (ping 안 됨, `running`) | `vagrant reload <머신 이름>`. 안 되면 VirtualBox에서 강제 재시작 |
| node_exporter만 멈춤 | `ssh devops@<IP> 'sudo systemctl restart prometheus-node-exporter'` |
| 방화벽이 9100을 막음 (ci-01, db-01) | 대상의 ufw에 `192.168.56.41 → 9100` 허용이 있는지 확인. 없으면 해당 role(jenkins·mysql) 담당에게 알림. playbook을 다시 실행하면 복구됨 |
| 방화벽이 9100을 막음 (devops-server) | 호스트 ufw는 **수동 관리**(role 아님). `sudo ufw status`에 `9100/tcp ALLOW 192.168.56.41`이 없으면 Ansible 담당에게 알림. 복구 명령: `sudo ufw allow from 192.168.56.41 to any port 9100 proto tcp comment 'node-exporter (mon-01)'` |

## 복구 확인

- Prometheus에서 `up{job="node", instance="<머신 이름>"}`이 1.
- Slack에 `[RESOLVED] InstanceDown`이 옴.
