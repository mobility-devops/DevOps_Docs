# guides

영역별 **구축 가이드**. 처음부터 같은 환경을 다시 만들 수 있도록 순서대로 적는다.
원본은 노션 「구축 가이드」이고, 확정된 것만 옮긴다.

| 파일 | 영역 | 담당 |
|---|---|---|
| `vm.md` | VM(Vagrant)·서버 기본 설정 | 김현서 |
| `backend.md` | 백엔드 실행·DB 준비 | 김현서 |
| `frontend.md` | 프론트엔드 실행·배포 | 임류경 |
| `ci.md` | Jenkins, SonarQube Cloud, 이미지 빌드 | 방지우 |
| `security.md` | Tailscale, SSH, fail2ban, Sealed Secrets, Trivy | 방지우 |
| `cd.md` | Argo CD, Argo Rollouts, GitOps 저장소 | 임류경 |
| `monitoring.md` | Prometheus, Grafana, Loki, Alertmanager | 손지원 (Lead) |

- 담당은 노션 「Team Member」의 역할 기준이다. 역할이 바뀌면 이 표와 문서 맨 위를 고친다.
- 각 가이드는 **사전 조건 → 설치 → 확인 방법** 순서로 쓴다.
