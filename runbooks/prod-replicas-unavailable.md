# ProdReplicasUnavailable

> 담당: 손지원 · 기준일: 2026-10-08 · 원본: [노션 「모니터링 경보 대응 Runbook」](https://app.notion.com/p/3f33f39c862e8087a141ee4393080dd8)

| 경보 | 등급 | 조건 |
| --- | --- | --- |
| ProdReplicasUnavailable | Critical (`#alert-critical`) | taxi-prod에 Ready 상태인 앱 Pod가 1분 동안 하나도 없음 (kube-state-metrics) |

## 증상

- `#alert-critical`에 `[FIRING] ProdReplicasUnavailable`이 옴.
- 보통 [prod-endpoint-down](prod-endpoint-down.md)이 함께 옴.

## 영향

- prod 서비스 중단.

## 확인

```bash
# 1. Pod 상태와 이유
kubectl -n taxi-prod get pods -o wide
kubectl -n taxi-prod describe pod <Pod 이름> | tail -30
kubectl -n taxi-prod logs <Pod 이름> --tail=100
kubectl -n taxi-prod logs <Pod 이름> --previous --tail=100     # 재시작했으면 직전 로그

# 2. 배포 상태
kubectl argo rollouts get rollout taxi -n taxi-prod

# 3. 노드 (Pod 를 놓을 곳이 있는지)
kubectl get nodes
```

| 보이는 것 | 뜻 |
| --- | --- |
| `CrashLoopBackOff`, 로그에 DB 접속 오류 | DB 문제 → [mysql-down](mysql-down.md) |
| `CrashLoopBackOff`, 그 밖의 오류 | 앱 오류. 방금 배포한 버전이면 롤백 |
| `ImagePullBackOff` | 이미지 주소·digest 오류. CI·CD 담당 |
| `Pending` | 노드 부족 → [multiple-workers-not-ready](multiple-workers-not-ready.md) |
| Running인데 Ready 아님 | readiness(`/actuator/health/readiness`) 실패. 로그 확인 |
| `OOMKilled` | 메모리 부족. 리소스 조정은 GitOps PR |

## 조치

| 원인 | 조치 |
| --- | --- |
| 방금 배포한 버전 | [GitOps 「prod 롤백 Runbook」](https://github.com/mobility-devops/DevOps_GitOps/blob/main/docs/runbook-rollback.md)의 되돌림 PR 절차를 따름. 되돌림 PR이 머지될 때까지 prod에 영향을 주는 다른 PR은 머지하지 않음 |
| DB·노드 문제 | 해당 Runbook을 먼저 처리하면 Pod가 스스로 Ready가 됨 |
| 설정(ConfigMap·Secret) 오류 | GitOps 저장소 수정 PR. 클러스터에서 손으로 고치면 Argo CD가 되돌림 |

## 복구 확인

- `kubectl -n taxi-prod get pods`의 앱 Pod가 모두 Running·Ready.
- Prometheus에서 `sum(kube_pod_status_ready{namespace="taxi-prod", condition="true"})`이 1 이상.
- Slack에 `[RESOLVED] ProdReplicasUnavailable`이 옴.
