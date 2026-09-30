# Git, Github Architecture

> 원본: 노션 「Git, Github Architecture」에서 2026-09-30 기준으로 옮김.

## Git Strategy

> **Hyundai AutoEver Mobility SW School 4th — DevOps Project**

---

## 1. Git Strategy 개요

프로젝트의 주요 Branch는 main과 develop으로 구성하며, 실제 개발 작업은 별도의 작업 Branch에서 수행

### 기본 Workflow

```text
Issue
  ↓
작업 Branch 생성
  ↓
개발
  ↓
Commit
  ↓
Push
  ↓
Pull Request
  ↓
Code Review + CI + CodeRabbit
  ↓
Merge
  ↓
develop
  ↓
통합 테스트
  ↓
Pull Request
  ↓
main
```

---

## 2. Branch Strategy

### 2.1 Branch 구조

```text
main
 │
 └── develop
       │
       ├── feature/*
       ├── fix/*
       ├── ci/*
       └── infra/*
```

### 2.2 Branch 역할

| Branch | 역할 |
|---|---|
| `main` | 배포 가능한 안정 버전 관리 |
| `develop` | 개발된 작업을 통합하는 Branch |
| `feature/*` | 새로운 기능 개발 |
| `fix/*` | 버그 수정 |
| `ci/*` | Jenkins, SonarQube 등 CI/CD 관련 작업 |
| `infra/*` | Kubernetes, Ansible 등 인프라 관련 작업 |

#### Branch Rules

- main에서 직접 개발 금지 → main은 배포 가능한 형태로만 유지
- develop에서 직접 기능을 개발 금지 → feature로 개발 후 merge
- 모든 개발 작업은 작업 Branch에서 수행
- 작업 Branch는 develop을 기준으로 생성
- 작업 완료 후 Pull Request를 통해 develop에 Merge
- 배포 가능한 상태가 되면 develop에서 main으로 Pull Request를 생성

---

## 3. Branch Naming Convention

#### 형식

```text
<type>/<issue-number>-<description>
```

#### 예시

```text
feature/12-login
feature/15-user
fix/21-login-error
ci/30-jenkins
infra/40-kubernetes
```

#### Branch Type

| Type | 용도 |
|---|---|
| `feature` | 새로운 기능 개발 |
| `fix` | 버그 수정 |
| `ci` | CI/CD 관련 작업 |
| `infra` | Kubernetes, Ansible 등 인프라 작업 |

---

## 4. Issue → Branch → PR 연결

```text
Issue #12
    ↓
feature/12-login
    ↓
Commit
    ↓
Pull Request
    ↓
develop
```

#### 예시

**Issue**

```text
[FEATURE] 로그인 API 구현
```

**Branch**

```text
feature/12-login
```

**Pull Request**

```text
feat: 로그인 API 구현
```

PR 본문에는 다음과 같이 관련 Issue를 연결

```text
Closes #12
```

이를 통해 다음 관계를 유지

```text
Issue #12
   ↕
Branch feature/12-login
   ↕
Pull Request
```

---

## 5. Development Workflow

### 5.1 작업 시작

먼저 `develop` Branch를 최신 상태로 업데이트

```bash
git switch develop
git pull origin develop
```

작업 Branch를 생성

```bash
git switch -c feature/12-login
```

현재 Branch를 확인

```bash
git branch
```

---

### 5.2 개발

생성한 작업 Branch에서 실제 개발을 진행

```bash
feature/12-login
```

필요한 경우 테스트 코드도 함께 작성

---

### 5.3 변경사항 확인

```bash
git status
```

전체 변경사항 확인

```bash
git diff
```

Staging된 변경사항 확인

```bash
git diff --staged
```

---

### 5.4 Commit

변경사항을 Stage에 추가

```bash
git add .
```

```bash
git commit -m "feat: 로그인 API 구현"
```

---

### 5.5 Push

처음 Push할 경우

```bash
git push -u origin feature/12-login
```

```bash
git push
```

---

## 6. Pull Request Strategy

Push가 완료되면 GitHub에서 Pull Request를 생성

#### 기능 개발

```text
feature/*
    ↓
Pull Request
    ↓
develop
```

#### 버그 수정

```text
fix/*
    ↓
Pull Request
    ↓
develop
```

#### CI/CD 작업

```text
ci/*
    ↓
Pull Request
    ↓
develop
```

#### Infrastructure 작업

```text
infra/*
    ↓
Pull Request
    ↓
develop
```

---

## 7. PR 수정 요청

Review 과정에서 수정 사항이 발생하면 새로운 PR을 생성하지 않음

기존 작업 Branch에서 수정한다.

```bash
git add .
git commit -m "fix: 로그인 예외 처리 수정"
git push
```

기존 PR에 새로운 Commit이 자동으로 추가

```text
PR #18
│
├── feat: 로그인 API 구현
└── fix: 로그인 예외 처리 수정
```

---

## 8. Merge Strategy

### 8.1 Feature → Develop

```text
feature/*
    ↓
Pull Request
    ↓
Code Review + CI
    ↓
develop
```

### 8.2 Develop → Main

```text
develop
    ↓
Pull Request
    ↓
Code Review + CI
    ↓
main
```

#### Merge 방식

본 프로젝트에서는 **Squash and Merge**를 기본 Merge 방식으로 사용

개발 과정에서 발생한 여러 Commit을 하나의 작업 단위로 정리하여 Merge

```text
feature/12-login

commit 1
commit 2
commit 3
commit 4
```

↓

```text
develop

feat: 로그인 API 구현
```

이를 통해 develop과 main의 Commit History를 작업 단위 중심으로 관리

---

## 9. Commit Convention

Commit 메시지는 다음 형식을 사용

```text
<type>: <description>
```

#### Type

| Type | 의미 |
|---|---|
| `feat` | 새로운 기능 |
| `fix` | 버그 수정 |
| `docs` | 문서 수정 |
| `refactor` | 기능 변경 없는 코드 구조 개선 |
| `test` | 테스트 코드 |
| `chore` | 기타 설정 및 작업 |
| `ci` | CI/CD 설정 |
| `build` | 빌드 관련 수정 |

#### 예시

```text
feat: 로그인 API 구현
fix: 로그인 예외 처리 수정
docs: Git Strategy 작성
refactor: 로그인 서비스 구조 개선
test: 로그인 API 테스트 추가
chore: Gradle 설정 변경
ci: Jenkins pipeline 추가
build: Docker 이미지 빌드 설정 추가
```

---

## 10. PR Title Convention

PR 제목도 Commit Convention과 동일한 형식을 사용

```text
<type>: <description>
```

#### 예시

```text
feat: 로그인 API 구현
fix: 로그인 예외 처리 수정
ci: Jenkins pipeline 추가
infra: Kubernetes deployment 구성
docs: Git Strategy 작성
```

---

## 11. CI 연계

CI는 **PR 단계**와 **develop merge 단계**로 나눠 실행 ([「아키텍처 개요」](../architecture/overview.md) 5장 기준)

- SonarQube Community는 PR별 분석을 지원하지 않아, SonarQube·Quality Gate는 develop merge 단계에서 실행
- Jenkins는 처음에 폴링으로 변경을 감지하고, 안정화 후 Webhook(Tailscale Funnel)으로 전환 💡

```text
[PR 단계]
Pull Request
      ↓
   Jenkins
      ↓
   Checkout
      ↓
Gradle Build / Test (Testcontainers MySQL)
      ↓
   JaCoCo 커버리지
      ↓
  PASS / FAIL  → PASS면 Code Review 후 Merge

[develop merge 단계]
develop merge
      ↓
Gradle Build / Test → JaCoCo
      ↓
  SonarQube → Quality Gate
      ↓
Docker 이미지 빌드 → Trivy 스캔
      ↓
  GHCR push (dev-짧은SHA)
```

#### Quality Gate

```text
Quality Gate
     │
 ┌───┴───┐
 ↓       ↓
PASS    FAIL
 ↓       ↓
PR      수정
검토    ↓
       재검증
```

Quality Gate(develop merge 단계)가 실패한 경우 원인을 확인하고, 수정 PR을 올려 다시 CI를 수행

---

## Template

GitHub Repository에서는 Issue Template을 사용하여 작업 등록 형식을 통일

Repository 구조:

```text
.github/
└── ISSUE_TEMPLATE/
    ├── feature_request.md
    ├── bug_report.md
    └── task.md
```

### Feature Request

````text
---
name: Feature Request
about: 새로운 기능 또는 기능 개선을 위한 작업
title: "[FEATURE] "
labels: "feature"
---

## 작업 목적

<!-- 이 기능이 필요한 이유를 작성 -->

## 작업 내용

- [ ]
- [ ]
- [ ]

## 완료 조건

- [ ]
- [ ]
- [ ]

## 참고 사항

<!-- 관련 문서, Architecture, Issue 등을 작성 -->
````

---

### Bug Report

````text
---
name: Bug Report
about: 오류 및 문제를 기록하고 수정하기 위한 작업
title: "[BUG] "
labels: "bug"
assignees: ""
---

## 버그 내용

<!-- 어떤 문제가 발생했는지 작성해 -->

## 재현 방법

1.
2.
3.

## 기대 결과

<!-- 정상적으로 동작해야 하는 결과 -->

## 실제 결과

<!-- 실제로 발생한 결과 -->

## 환경

- OS:
- Application:
- Branch:
- Version:

## 로그 / 에러

```text

```
````

---

### Pull Request Template

#### Template

````text
## 작업 내용

<!-- 이번 PR에서 변경한 내용을 작성 -->

-
-
-

## 관련 Issue

Closes #

## 변경 사항

### Application

- [ ] 기능 추가
- [ ] 버그 수정
- [ ] 리팩토링
- [ ] 테스트 추가

### CI/CD

- [ ] Jenkins
- [ ] SonarQube
- [ ] Docker
- [ ] GitOps
- [ ] Argo CD

### Infrastructure

- [ ] Kubernetes
- [ ] Argo Rollouts
- [ ] Ansible
- [ ] Monitoring

## 테스트

- [ ] Local Test
- [ ] Gradle Test
- [ ] Jenkins CI (Build · Test · JaCoCo)
- [ ] SonarQube Quality Gate (develop merge 후 확인)

### 테스트 결과

```text

```
````

---

## 12. 전체 Git Workflow

```text
                GitHub
                   │
                   ↓
                 Issue
                   │
                   ↓
            작업 Branch 생성
                   │
    ┌──────────────┼──────────────┐
    ↓              ↓              ↓
 feature/*       fix/*          infra/*
    │              │              │
    └──────────────┼──────────────┘
                   ↓
                  개발
                   │
                   ↓
          add → commit → push
                   │
                   ↓
            Pull Request
                   │
      ┌────────────┴────────────┐
      ↓                         ↓
Code Review                  Jenkins CI
                                │
                         ┌──────┼──────┐
                         ↓      ↓      ↓
                      Build   Test   JaCoCo
                                       │
                                       ↓
                                  CI 결과
                                       │
                   ┌───────────────────┴───────────────────┐
                   ↓                                       ↓
                 PASS                                    FAIL
                   ↓                                       ↓
              PR Merge                                  수정
                   ↓                                       │
                develop ←──────────────────────────────────┘
                   │
                   ↓
               통합 테스트
                   │
                   ↓
            develop → main
                   │
                   ↓
                 main
                   │
                   ↓
                  배포
```

※ develop merge 후에는 SonarQube · Quality Gate → 이미지 빌드 → Trivy → GHCR push → gitops 갱신(dev 자동 배포)이 이어짐. main에는 릴리스 태그(`v1.0.0`)를 붙이고, prod는 gitops PR 승인 후 Canary로 배포 (「아키텍처 개요」 5·6·10장).
