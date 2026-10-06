# DevOps_Docs

현대오토에버 모빌리티 SW 스쿨 4기 DevOps 프로젝트(택시 배차 서비스)의 **확정 문서 저장소**.
팀이 확정한 내용만 둔다. 초안과 논의는 노션에서 한다.

## 문서

| 문서 | 내용 | 기준일 |
|---|---|---|
| [프로젝트 아키텍처](architecture/project-architecture.md) | 목표 아키텍처 전체: 물리·VM 구성, 네트워크, CI·CD, 앱·DB, 모니터링, 보안, 저장소 규칙 | 2026-10-06 |

- 원본은 노션 「프로젝트 아키텍처」다. 서로 다르면 노션이 기준이다.
- 표기: ✅ 결정 · 💡 제안(회의에서 확인) · ❓ 미정

## 규칙

- 노션에서 확정된 내용이 바뀌면 이 저장소에도 반영한다(기준일 갱신).
- 새 문서는 팀이 확정한 것만 추가한다.
- 작업 브랜치 → `main` 대상 PR(승인 1명) → Squash and merge.

## 관련 저장소

| 저장소 | 역할 |
|---|---|
| [DevOps_Backend](https://github.com/mobility-devops/DevOps_Backend) | 앱 코드, Dockerfile, Jenkinsfile |
| [DevOps_GitOps](https://github.com/mobility-devops/DevOps_GitOps) | 배포 상태(Kustomize, Argo CD) |
| [DevOps_Infra](https://github.com/mobility-devops/DevOps_Infra) | VM(Vagrant)과 서버 설정 |
| **DevOps_Docs** (이 저장소) | 확정 문서 |
