# MultipleWorkersNotReady

> 담당: 손지원 · 기준일: 2026-10-08 · 원본: [노션 「모니터링 경보 대응 Runbook」](https://app.notion.com/p/3f33f39c862e8087a141ee4393080dd8)

| 경보 | 등급 | 조건 |
| --- | --- | --- |
| MultipleWorkersDown | Critical (`#alert-critical`) | worker 2대 이상의 node_exporter가 1분 동안 응답 없음 |
| MultipleWorkersNotReady | Critical (`#alert-critical`) | worker 2대 이상이 1분 동안 NotReady (kube-state-metrics) |

- 두 경보는 같은 상황을 다른 쪽에서 본 것임. MultipleWorkersDown이 울리면 MultipleWorkersNotReady는 Slack에 보내지 않음(중복 억제).

## 증상

- `#alert-critical`에 위 경보 중 하나가 옴.
- 남은 worker 1대로 운영 중이라는 뜻임.

## 영향

- 남은 worker 1대에 모든 Pod가 몰림. 메모리가 부족하면 Pod가 Pending으로 남음.
- 남은 1대까지 빠지면 prod가 멈춤. **가장 먼저 복구할 경보임.**

## 확인

```bash
# 1. 노드 상태 (kubeconfig 가 있는 곳에서)
kubectl get nodes -o wide
kubectl get pods -A -o wide --field-selector=status.phase!=Running

# 2. 어느 머신 문제인지: worker1·2 는 호스트 PC, worker3 은 노트북
cd ~/DevOps_Infra && vagrant status k8s-worker1 k8s-worker2    # 호스트 PC
# worker3 은 노트북에서: vagrant status k8s-worker3

# 3. 노드는 켜져 있는데 NotReady 면 kubelet 확인
ssh devops@<worker IP> 'systemctl status kubelet --no-pager; sudo journalctl -u kubelet -n 50 --no-pager'
```

## 조치

| 원인 | 조치 |
| --- | --- |
| 노트북 꺼짐·랜선 빠짐 + 호스트 worker 1대 장애 | 먼저 [laptop-down](laptop-down.md) 순서로 노트북을 살리고, 호스트 worker는 [instance-down](instance-down.md) 순서로 살림 |
| 호스트 worker 2대 모두 꺼짐 | `vagrant up k8s-worker1 k8s-worker2` |
| VM은 켜져 있는데 kubelet 오류 | `ssh devops@<IP> 'sudo systemctl restart kubelet'`. 반복되면 kubelet 로그를 클러스터 담당에게 전달 |
| 남은 worker 메모리 부족으로 Pending | 복구 전까지 prod 우선. dev(`taxi-dev`)를 줄이는 것은 CD 담당과 협의 |

## 복구 확인

- `kubectl get nodes`에서 worker 3대가 Ready (노트북을 일부러 끈 시연이면 2대).
- `kubectl get pods -n taxi-prod`의 Pod가 모두 Running·Ready.
- Slack에 `[RESOLVED]`가 옴.
