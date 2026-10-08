# ClusterMetricsMissing

> 담당: 손지원 · 기준일: 2026-10-08 · 원본: [노션 「모니터링 경보 대응 Runbook」](https://app.notion.com/p/3f33f39c862e8087a141ee4393080dd8)

| 경보 | 등급 | 조건 |
| --- | --- | --- |
| ClusterMetricsMissing | Warning | 5분 동안 kube-state-metrics 지표가 들어오지 않음: `absent_over_time(up{job="kube-state-metrics"}[5m])` |

## 증상

- `#alert-warning`에 `[FIRING] ClusterMetricsMissing`이 옴.
- 클러스터 지표는 Alloy가 모아서 mon-01로 보냄(remote_write). 이 경로 어딘가가 끊긴 것임.

## 영향

- 서비스에는 영향 없음. 하지만 **클러스터 경보(Worker3NotReady, MultipleWorkersNotReady, ProdReplicasUnavailable)가 울리지 못함.**
- 앱 지표도 같은 경로라 Canary 판정이 Inconclusive(판단 불가)로 멈출 수 있음.

## 확인

```bash
# 1. Argo CD 에서 수집 도구 상태
kubectl -n argocd get applications platform-alloy platform-kube-state-metrics

# 2. Pod 상태
kubectl -n monitoring get pods -o wide

# 3. Alloy 로그 (전송 오류)
kubectl -n monitoring logs ds/alloy --tail=50 | grep -i -E "error|fail"

# 4. Pod 에서 mon-01 로 가는 길 (Calico 정책·mon-01 ufw)
kubectl -n monitoring run nettest --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://192.168.56.41:9090/-/ready
```

## 조치

| 원인 | 조치 |
| --- | --- |
| Application OutOfSync·Degraded | Argo CD 화면에서 오류 메시지 확인. GitOps `platform/alloy`·`platform/kube-state-metrics` 최근 변경을 되돌림 |
| kube-state-metrics Pod 재시작 반복 | `kubectl -n monitoring describe pod <이름>`에서 메모리 부족(OOMKilled)인지 봄. 리소스 조정은 GitOps PR로 함 |
| Alloy 전송 실패 (connection refused·timeout) | mon-01 Prometheus가 살아 있는지 확인. mon-01 ufw에 노드 IP(.21~.24) → 9090 허용 확인 |
| 4번에서 Pod → mon-01 실패 | Calico 전역 정책(CD 담당, GitOps `platform/`)이 막는지 CD 담당과 함께 확인 |

- 클러스터에서 손으로 고친 것은 Argo CD selfHeal이 되돌림. 고칠 것은 GitOps 저장소 PR로 함.

## 복구 확인

- Prometheus에서 `up{job="kube-state-metrics"}`가 1, `kube_pod_info`가 조회됨.
- Slack에 `[RESOLVED] ClusterMetricsMissing`이 옴.
