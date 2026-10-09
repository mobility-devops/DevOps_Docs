# runbooks

장애 대응·운영 절차. Alertmanager 알림마다 대응 문서를 하나씩 둔다(아키텍처 8장).

- 파일 이름은 알림 이름을 따른다. 예: `node-down.md`, `laptop-link-down.md`
- 알림의 `runbook_url`에 이 문서 링크를 넣는다.

## 문서

### 모니터링 경보 (담당: 손지원)

| 문서 | 경보 | 등급 | 켜지는 시점 |
| --- | --- | --- | --- |
| [instance-down](instance-down.md) | InstanceDown | Warning | mon-01 배포 |
| [laptop-down](laptop-down.md) | LaptopLinkDown, LaptopTargetDown, Worker3NotReady | Warning | mon-01 배포 (Worker3NotReady는 클러스터 경보) |
| [multiple-workers-not-ready](multiple-workers-not-ready.md) | MultipleWorkersDown, MultipleWorkersNotReady | Critical | mon-01 배포 (NotReady는 클러스터 경보) |
| [disk-almost-full](disk-almost-full.md) | DiskAlmostFull | Warning·Critical | mon-01 배포 |
| [mysql-down](mysql-down.md) | MySQLDown | Critical | mon-01 배포 |
| [watchdog-missing](watchdog-missing.md) | healthchecks.io `alertmanager-watchdog` DOWN | Critical | mon-01 배포 |
| [cluster-metrics-missing](cluster-metrics-missing.md) | ClusterMetricsMissing | Warning | 클러스터 경보 켤 때 |
| [prod-endpoint-down](prod-endpoint-down.md) | ProdEndpointDown | Critical | 앱 경보 켤 때 |
| [prod-replicas-unavailable](prod-replicas-unavailable.md) | ProdReplicasUnavailable | Critical | 앱 경보 켤 때 |

- 배포 알림(Argo CD, `#taxi-deploy`)의 대응은 GitOps 저장소의 [prod 롤백 Runbook](https://github.com/mobility-devops/DevOps_GitOps/blob/main/docs/runbook-rollback.md)을 따른다.

## 형식

```markdown
# 알림 이름

> 담당: 이름 · 기준일: YYYY-MM-DD · 원본: [노션 「문서 이름」](노션 링크)

## 증상
## 영향
## 확인
## 조치
## 복구 확인
```
