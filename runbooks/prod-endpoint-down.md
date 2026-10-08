# ProdEndpointDown

> 담당: 손지원 · 기준일: 2026-10-08 · 원본: [노션 「모니터링 경보 대응 Runbook」](https://app.notion.com/p/3f33f39c862e8087a141ee4393080dd8)

| 경보 | 등급 | 조건 |
| --- | --- | --- |
| ProdEndpointDown | Critical (`#alert-critical`) | blackbox가 1분마다 Gateway(`192.168.56.200`)를 거쳐 prod health를 호출하는데, 2분 동안 실패: `probe_success{job="blackbox-prod"} == 0` |

## 증상

- `#alert-critical`에 `[FIRING] ProdEndpointDown`이 옴.
- 사용자가 들어오는 길(Gateway → HTTPRoute → 앱 Pod)을 밖에서 확인한 결과라, **사용자도 접속하지 못하는 상태**일 가능성이 큼.

## 영향

- prod 서비스 중단. 가장 먼저 대응함.

## 확인

어느 구간에서 끊겼는지 앞에서부터 확인함.

```bash
# 1. 밖에서 직접 호출 (호스트 PC 또는 Tailscale 접속 PC)
curl -sv -m 5 -H "Host: <prod 이름>" http://192.168.56.200<health 경로>

# 2. 앱 Pod 와 Rollout
kubectl -n taxi-prod get pods -o wide
kubectl argo rollouts get rollout taxi -n taxi-prod

# 3. Gateway (Envoy) 와 MetalLB
kubectl get gateway -A
kubectl -n envoy-gateway-system get pods -o wide
kubectl -n metallb-system get pods -o wide

# 4. 경보 자체가 틀렸는지 (mon-01 의 blackbox 결과)
#    Prometheus 화면: probe_http_status_code{job="blackbox-prod"}, probe_duration_seconds{job="blackbox-prod"}
```

| 결과 | 뜻 | 다음 |
| --- | --- | --- |
| 2번에서 Ready Pod가 없음 | 앱 문제 | [prod-replicas-unavailable](prod-replicas-unavailable.md) |
| Pod는 Ready인데 1번이 실패 | Gateway·MetalLB·HTTPRoute 문제 | 3번 결과를 CD 담당과 확인 |
| 1번은 성공하는데 경보만 울림 | blackbox 설정 문제 (주소·이름·인증서) | 아래 조치의 마지막 줄 |

## 조치

| 원인 | 조치 |
| --- | --- |
| 방금 배포한 버전이 원인 | [GitOps 「prod 롤백 Runbook」](https://github.com/mobility-devops/DevOps_GitOps/blob/main/docs/runbook-rollback.md)의 되돌림 PR 절차를 따름 |
| Envoy·MetalLB Pod 이상 | Argo CD에서 `platform-envoy-gateway`, `platform-metallb` 상태 확인. CD 담당과 함께 처리 |
| DB 문제로 health 실패 | [mysql-down](mysql-down.md) |
| blackbox 설정이 실제 주소와 다름 | DevOps_Infra `group_vars/monitoring/main.yml`의 `prod_probe_host`·`prod_probe_path`를 고치는 PR. 그동안 Alertmanager에서 잠시 끄기(silence) |

## 복구 확인

- 1번 호출이 200.
- Prometheus에서 `probe_success{job="blackbox-prod"}`가 1.
- Slack에 `[RESOLVED] ProdEndpointDown`이 옴.
