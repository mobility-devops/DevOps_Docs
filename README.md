# DevOps_Docs

현대오토에버 모빌리티 SW 스쿨 4기 DevOps 프로젝트(택시 배차 서비스)의 설계·운영 문서 저장소.

- 표기: ✅ 결정 · 💡 제안(회의에서 확인) · ❓ 미정
- 노션 문서를 2026-09-30 기준으로 옮김. 이후 인프라나 코드를 바꾸는 PR에서 관련 문서도 함께 수정.
- 문서 추가·수정은 작업 브랜치 → `main` 대상 PR → Squash and merge.

> ⚠️ **아래 문서는 2026-09-30 기준이다.** 10/4~10/5에 아키텍처가 바뀌었으므로(노트북 서버·worker3 추가, br-lab,
> mon-01, Gateway API, SonarQube Cloud, webhook, Wazuh·controlnode·sonar-01 제외 등) 최신 설계는 노션 「프로젝트 아키텍처」와
> 그 사본 [DevOps_Infra/docs/architecture.md](https://github.com/mobility-devops/DevOps_Infra/blob/main/docs/architecture.md)를 본다.

## 문서 목록

### 아키텍처 (`architecture/`)

| 문서 | 내용 |
|---|---|
| [아키텍처 개요](architecture/overview.md) | 목표 아키텍처, VM·네트워크, CI/CD, 모니터링, 보안, 자원 운영 방안 |
| [쉽게 이해하는 아키텍처](architecture/overview-easy.md) | 아키텍처 개요를 쉬운 말과 비유로 풀어쓴 해설 |
| [Backend 설계](architecture/backend.md) | 택시 배차 서비스 백엔드 전체 설계와 v1 구현 범위 |
| [현재 환경 현황](architecture/current-environment.md) | 호스트 PC, VM, 네트워크의 현재 상태 |

### 로드맵 (`roadmap/`)

| 문서 | 내용 |
|---|---|
| [로드맵과 할 일](roadmap/plan.md) | 0~7단계 작업, 완료 기준, 결정 현황, 리스크 |
| [쉽게 이해하는 로드맵](roadmap/plan-easy.md) | 로드맵을 쉬운 말과 비유로 풀어쓴 해설 |

### 규칙 (`conventions/`)

| 문서 | 내용 |
|---|---|
| [Git 전략](conventions/git-strategy.md) | 브랜치·커밋·PR 규칙, Issue/PR 템플릿, CI 연계 |

### 회의 기록 (`meetings/`)

| 날짜 | 문서 |
|---|---|
| 2026-09-24 | [회의 기록](meetings/2026-09-24.md) |
| 2026-09-28 | [회의 기록](meetings/2026-09-28.md) · [Port](meetings/2026-09-28-port.md) · [서버 초기 구축](meetings/2026-09-28-server-setup.md) · [Server GUI 작업 정리](meetings/2026-09-28-server-gui.md) |
| 2026-09-29 | [회의 기록](meetings/2026-09-29.md) |
| 2026-09-30 | [회의 기록](meetings/2026-09-30.md) |

## 관련 저장소

| 저장소 | 역할 |
|---|---|
| [DevOps_Backend](https://github.com/mobility-devops/DevOps_Backend) | Spring Boot 코드, Dockerfile, Jenkinsfile |
| [DevOps_GitOps](https://github.com/mobility-devops/DevOps_GitOps) | 배포 상태(Kustomize, Argo CD 설정) |
| [DevOps_Infra](https://github.com/mobility-devops/DevOps_Infra) | VM(Vagrant)과 서버 설정 |
| **DevOps_Docs** (이 저장소) | 설계·운영 문서, ADR, Runbook |
