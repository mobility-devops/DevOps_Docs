# LaptopDown

> 담당: 손지원 · 기준일: 2026-10-08 · 원본: [노션 「모니터링 경보 대응 Runbook」](https://app.notion.com/p/3f33f39c862e8087a141ee4393080dd8)

| 경보 | 등급 | 조건 |
| --- | --- | --- |
| LaptopLinkDown | Warning | 호스트 PC 유선 랜포트 링크가 30초 동안 끊김: `node_network_carrier{instance="devops-server", device="<유선 랜포트>"} == 0` |
| LaptopTargetDown | Warning | 노트북(laptop-01) 또는 worker3의 node_exporter가 1분 동안 응답 없음 |
| Worker3NotReady | Warning | worker3이 1분 동안 NotReady (kube-state-metrics) |

- 노트북은 호스트 PC와 랜선 한 줄로만 연결됨. LaptopLinkDown이 울리면 나머지 두 경보는 Slack에 보내지 않음(중복 억제).
- 노트북 장애 시연 중이면 예상된 경보임. 시연 전에 Alertmanager에서 잠시 끄기(silence)를 걸어 둠.

## 증상

- `#alert-warning`에 위 경보 중 하나가 옴. 가장 먼저 오는 것은 보통 LaptopLinkDown임.

## 영향

- worker3만 빠짐. prod는 호스트의 worker 2대가 이어받아 계속 응답함.
- 머신마다 1개씩 뜨던 Pod(Envoy 프록시, prod 앱)가 호스트 쪽에만 남음.
- 이 상태에서 호스트 worker 1대가 더 빠지면 [multiple-workers-not-ready](multiple-workers-not-ready.md)가 됨.

## 확인

```bash
# 1. 랜선·노트북 응답 (호스트 PC에서)
ip link show <유선 랜포트>          # state UP 이어야 함. NO-CARRIER 면 랜선·노트북 문제
ping -c 3 192.168.56.2              # 노트북
ping -c 3 192.168.56.24             # worker3

# 2. 노트북에 접속되면 VM 상태 (노트북에서)
vagrant status k8s-worker3

# 3. 클러스터에서 worker3
kubectl get node k8s-worker3
```

## 조치

| 원인 | 조치 |
| --- | --- |
| 랜선 빠짐 | 양쪽 랜포트에 다시 꽂음. 링크 표시등 확인 |
| 노트북 꺼짐·절전 | 전원을 켬. 덮개를 닫아도 꺼지지 않는 설정(lid 무시)이 풀렸는지 확인 |
| 노트북은 켜졌는데 worker3 VM이 꺼짐 | 노트북에서 `vagrant up k8s-worker3` |
| VM은 켜졌는데 NotReady | `ssh devops@192.168.56.24 'sudo systemctl restart kubelet'` |

## 복구 확인

- `ip link show <유선 랜포트>`가 `state UP`.
- `kubectl get node k8s-worker3`이 Ready.
- Prometheus에서 `up{job="node", zone="laptop"}`이 모두 1.
- Slack에 `[RESOLVED]`가 옴.
