# DevOps_Docs

현대오토에버 모빌리티 SW 스쿨 4기 DevOps 프로젝트(택시 배차 서비스)의 **확정 문서 저장소**.
팀이 확정한 내용만 둔다. 초안과 논의는 노션에서 한다.

## 폴더 구조

```text
architecture/   프로젝트 아키텍처 (목표 구조 전체)
guides/         구축 가이드 (영역별: vm, backend, frontend, ci, security, cd, monitoring)
runbooks/       장애 대응·운영 절차 (알림별 대응 문서)
adr/            주요 결정 기록 (무엇을 왜 그렇게 정했는지)
```

사람이 아니라 **주제별**로 나눈다. 누가 맡았는지는 각 문서 맨 위에 적는다.

## 문서

| 문서 | 내용 | 기준일 |
|---|---|---|
| [프로젝트 아키텍처](architecture/project-architecture.md) | 목표 아키텍처 전체: 물리·VM 구성, 네트워크, CI·CD, 앱·DB, 모니터링, 보안, 저장소 규칙 | 2026-10-06 |

- `guides/`, `runbooks/`, `adr/`의 문서는 노션에서 확정되는 대로 추가한다.
- 원본은 노션이다. 서로 다르면 노션이 기준이다.
- 표기: ✅ 결정 · 💡 제안(회의에서 확인) · ❓ 미정

## 문서 작성 형식

모든 문서는 맨 위에 담당자, 기준일, 노션 원본을 적는다.

```markdown
# 문서 제목

> 담당: 이름 · 기준일: YYYY-MM-DD · 원본: [노션 「문서 이름」](노션 링크)
```

- 파일 이름은 영어 소문자와 `-`로 쓴다. 예: `guides/ci.md`, `runbooks/node-down.md`
- 그림은 같은 폴더의 `images/`에 둔다.

## 노션에서 옮기는 절차

1. 노션에서 문서를 확정한다(회의 또는 담당자 확인).
2. 이슈를 만든다. 제목 `[Docs] 설명`, 근거에 노션 링크를 적는다.
3. 최신 `main`에서 `docs/<이슈번호>-<설명>` 브랜치를 만든다.
4. 위 형식대로 문서를 추가하고, 이 README의 문서 표에 한 줄 추가한다.
5. PR(제목 `docs: 설명`, 본문 `Closes #번호`) → 승인 1명 → **Squash and merge**.

노션에서 확정된 내용이 바뀌면 이 저장소에도 반영하고 기준일을 갱신한다.

## 규칙

- 확정된 문서만 올린다. 초안은 노션에 둔다.
- 비밀번호·토큰·비밀키는 적지 않는다(저장소 public).
- `main` 하나. `main`에 직접 push하지 않는다.

## 관련 저장소

| 저장소 | 역할 |
|---|---|
| [DevOps_Backend](https://github.com/mobility-devops/DevOps_Backend) | 앱 코드, Dockerfile, Jenkinsfile |
| [DevOps_GitOps](https://github.com/mobility-devops/DevOps_GitOps) | 배포 상태(Kustomize, Argo CD) |
| [DevOps_Infra](https://github.com/mobility-devops/DevOps_Infra) | VM(Vagrant)과 서버 설정 |
| **DevOps_Docs** (이 저장소) | 확정 문서 |
