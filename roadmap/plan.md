# 로드맵과 할 일

> ⚠️ **9/30 기준 문서라 지금 설계와 다르다.** 10/1~10/5에 아키텍처가 바뀌었다(노트북 서버·worker3 추가, br-lab, mon-01,
> Gateway API, SonarQube Cloud, webhook, Wazuh·controlnode·sonar-01 제외 등). 최신 설계는 [프로젝트 아키텍처](../architecture/overview.md)를 본다.

> 원본: 노션 「로드맵과 할 일」에서 2026-09-30 기준으로 옮김.

> 📌 **상태: 회의 전 초안** · 기준일 2026-09-30 · 담당자와 일정은 회의에서 결정.
> 범례: ✅ 결정 · 💡 제안(회의에서 확인) · ❓ 미정

## 1. 진행 현황

| 트랙 | 완료 | 남음 |
|---|---|---|
| 앱 (backend Git 이력 기준) | v1 기능 전체(develop 병합, #2~#20): 초기 설정, 공통 응답·예외, 사용자, 기사·차량, 호출 생성·조회·수락·운행 상태 변경·취소, Swagger, 전체 흐름 통합 테스트 | 배포 준비: DB 접속 URL 환경변수화(현재 `localhost` 고정), Actuator·Micrometer, Dockerfile, Jenkinsfile, Testcontainers·JaCoCo, Canary용 v2 |
| 인프라 | 환경 조사, 설계 결정, DevOps_Docs main 브랜치와 PR 생성, DevOps_Infra 저장소 생성 | 결정 확정, 1~7단계 |

## 2. 단계별 계획

### 0단계. 준비 (진행 중)

- [ ] 이 문서를 팀 회의에서 검토하고 문서 원본 위치 결정 (노션 유지 또는 docs 저장소 이전 ❓)
- [ ] VM 자원 조정안(합계 41GB + 단계별 기동) 팀 합의 💡 — [「아키텍처 개요」](../architecture/overview.md) 3장 「자원 운영 방안」
- [ ] Wazuh 도입 여부와 범위 결정 💡 — 「아키텍처 개요」 9장 「Wazuh 보안 감시」

> 🎯 **완료 기준:** 1단계 착수에 필요한 결정(SSH 키, 로그인 사용자명 등)이 확정된 상태.

### 1단계. VM 기반

- [ ] 호스트에 Vagrant 설치 확인 (VirtualBox 7.2 호환 확인)
- [ ] 기존 VM 정리 (managednode1·2, ubuntu-server-02 폴더)
- [ ] Vagrantfile로 신규 VM 정의 (고정 IP, KST, swap 해제, 공개키 배포). 신규 6대 + Wazuh 도입 시 sec-01 💡, 스펙은 자원 조정안 기준 💡
- [ ] 1단계에서는 k8s 3대와 db-01만 기동(`autostart: true`). ci-01·sonar-01·sec-01은 `autostart: false`로 두고 필요한 단계에서 기동 💡
- [ ] controlnode(VirtualBox 이름 `ubuntu-server-01`) RAM을 4GB에서 2GB로 축소 💡
- [ ] controlnode에 ansible-core 설치
- [ ] Ansible `common` role 작성 (chrony, SSH 하드닝, fail2ban, 호스트 이름 해석)
- [ ] 호스트 Tailscale Subnet Router 설정 (192.168.56.0/24 광고)

**산출물:** DevOps_Infra의 Vagrantfile, Ansible 인벤토리, roles/common

> 🎯 **완료 기준:** `vagrant up`으로 기동 대상 VM(k8s 3대, db-01)이 켜지고, controlnode에서 `ansible all -m ping`이 성공하며, 모든 VM이 KST·swap off·SSH 키 접속 상태. 나머지 VM도 `vagrant up <이름>`으로 기동 확인.

### 2단계. Kubernetes와 DB

- [ ] containerd·kubeadm 설치 role, 클러스터 초기화 (노드 IP는 Host-Only로 명시)
- [ ] Calico(Pod CIDR 10.244.0.0/16), MetalLB, Ingress-NGINX, cert-manager(자체 CA), local-path-provisioner, metrics-server(HPA용)
- [ ] 노드 라벨(worker1 `node-role/platform`, worker2 `node-role/monitoring`), master taint 유지
- [ ] MySQL 8 role (db-01): 스키마와 계정, 방화벽(3306은 worker 2대와 호스트만), 백업 스크립트, mysqld_exporter, `innodb_buffer_pool_size` 1GB 💡

> 🎯 **완료 기준:** `kubectl get nodes`에서 3대 Ready, 샘플 앱이 Ingress로 HTTPS 응답, worker에서 db-01의 3306 접속 성공(허용되지 않은 IP는 거부), 백업 복구 리허설 1회.

### 3단계. CI

- [ ] ci-01·sonar-01 기동 (`vagrant up ci-01 sonar-01`)
- [ ] ci-01에 Jenkins(JCasC), sonar-01에 SonarQube 설치. 메모리 상한: Jenkins `-Xmx1g`, Gradle `org.gradle.jvmargs=-Xmx1g`, sonar-01 `vm.max_map_count` 설정 💡
- [ ] GitHub App과 GHCR 토큰 생성·등록
- [ ] 파이프라인 작성: PR은 테스트 → JaCoCo, develop merge는 여기에 SonarQube → 이미지 빌드 → Trivy → GHCR push (처음엔 폴링)
- [ ] backend에 Dockerfile(멀티스테이지, non-root)과 Jenkinsfile 추가

**전제:** 앱이 빌드·테스트 가능한 상태여야 함.

> 🎯 **완료 기준:** develop에 merge하면 이미지가 GHCR에 자동으로 올라가고, Quality Gate 실패 시 파이프라인이 중단됨.

### 4단계. GitOps와 CD

- [ ] gitops 저장소 구조 작성 (Kustomize base와 overlays)
- [ ] Argo CD 설치, App of Apps, taxi-dev / taxi-prod namespace, Sealed Secrets
- [ ] Jenkins의 gitops 갱신: dev는 직접 push, prod는 PR 생성

> 🎯 **완료 기준:** develop merge 후 dev에 자동 배포되고, prod는 PR 승인 후에만 배포되며, 커밋을 되돌려 롤백 가능.

### 5단계. 모니터링과 알림

- [ ] kube-prometheus-stack, ServiceMonitor (Prometheus 보존 기간은 짧게, 예: 3~7일 💡)
- [ ] Loki + Promtail (메모리 여유를 확인한 뒤 이 단계 마지막에 도입 💡)
- [ ] Grafana 대시보드(SLO 포함), Slack 알림 (정책 확정 후), Runbook 초안

> 🎯 **완료 기준:** 앱·DB·노드 지표가 대시보드에 표시되고 테스트 알림이 Slack에 도착함.

### 6단계. Canary 배포

- [ ] Deployment를 Rollout으로 전환, AnalysisTemplate(에러율, p95), Smoke Test Job, 부하 생성

> 🎯 **완료 기준:** 정상 버전은 10% → 100%로 자동 진행되고, 의도적으로 오류를 낸 버전은 자동으로 롤백됨(시연).

### 7단계. 운영 마무리

- [ ] 백업 자동화와 외부 복사, 복구 리허설 정례화
- [ ] RBAC, Pod Security, NetworkPolicy(앱 Pod만 db-01:3306으로 나가도록 egress 제한 포함), ResourceQuota 적용
- [ ] Wazuh 도입 시 💡: sec-01 기동, Wazuh Manager·Indexer·Dashboard 설치, 전 VM에 agent 설치(Ansible), K8s 감사 로그 연동, Slack 보안 채널 연결, 시연 시나리오 1~2건
- [ ] Webhook(Tailscale Funnel) 사용 시 노출 대응: webhook 경로만 공개, webhook secret 검증, Jenkins 익명 권한 제거 💡
- [ ] 장애 시나리오 시연 (DB 다운, 배포 롤백)
- [ ] ADR과 Runbook 정리, 최종 문서 갱신 후 docs 저장소 반영

> 🎯 **완료 기준:** 복구 리허설을 통과하고, 장애 시연 2건을 완료하고, 문서가 최신 상태.

**앱 트랙(병행):** v1 기능은 develop에 모두 병합되어 앱은 빌드·테스트 가능한 상태. 남은 것은 배포 준비 작업(1장 표). 3단계 CI 전에 DB URL 환경변수화와 Dockerfile, 5단계 전에 Actuator·Micrometer, 6단계 전에 Canary용 v2를 준비.

## 3. 결정 현황

### 결정 대기

| 항목 | 추천 | 비고 |
|---|---|---|
| VM에 넣을 SSH 공개키 | 전용 키를 새로 생성 | 공개키만 공유하고 비밀키는 공유 금지 |
| VM 로그인 사용자명, sudo | 사용자명 devops, sudo 비밀번호 없음(NOPASSWD) | Ansible 자동 실행에 필요 |
| controlnode에서 새 VM으로 접속하는 키 방식 | controlnode에 별도 키를 만들어 배포 | Vagrant가 공개키를 심음 |
| DevOps_Infra 브랜치 규칙 | docs와 동일(main + 작업 브랜치, PR, Squash and merge) | main 브랜치를 먼저 만들어야 함 |
| 인프라 작업 GitHub Issue 초안 | 작성 | Issue 생성은 사용자가 직접 |
| 노션·`CLAUDE.md` 갱신 시점 | VM·K8s가 뜬 뒤 한 번에 | `CLAUDE.md`의 설계 원본 문구 포함 |
| 백엔드와 인프라의 진행 순서 | 인프라 1~2단계를 먼저 | 앱은 3단계에서 필요 |

### 나중에 정할 것

| 항목 | 상태 |
|---|---|
| 팀 역할 분담 | 팀 노션(09-29)에 초기 담당자 있음, 회의에서 확인 |
| 알림 정책 (Slack 채널, 담당자) | ❓ 회의에서 결정 |
| hotfix 규칙 | ❓ 필요해질 때 추가 |
| 도메인 | 없음, 자체 CA로 시작 ✅ |
| Jenkins 트리거 방식 | 💡 폴링으로 시작, 이후 Webhook(Tailscale Funnel, 노출 대응 포함) |
| Wazuh 도입 여부와 범위 | 💡 별도 VM sec-01, agent는 전 VM, 7단계 보조 시연 |
| fail2ban 유지 여부 | ❓ Wazuh 도입 시 함께 결정 |
| VM 자원 조정안 | 💡 합계 41GB + 단계별 기동, 팀 합의 필요 |
| 문서 원본 위치 | ❓ 노션 유지 또는 DevOps_Docs 이전 |
| GitHub App·GHCR 토큰 생성 시점 | 💡 3단계 |
| 호스트 SSH 범위 축소 시점 | 💡 Tailscale 접속 확인 후 |
| Flyway 마이그레이션 방식 | 💡 앱 기동 시 자동, 하위 호환만 |
| Tailscale 팀원 접속 방식 | ❓ 팀 역할과 함께 |

## 4. 직접 해야 할 일 (사용자)

이 프로젝트를 돕는 Claude 세션은 클라우드에서 실행되어 호스트 PC, VirtualBox, 노션 삭제, GitHub 설정은 직접 해야 함.

- [ ] DevOps_Docs PR을 Squash and merge로 병합
- [ ] DevOps_Docs 기본 브랜치를 main으로 변경, docs/env-status 브랜치 삭제
- [ ] 노션의 옛 페이지 「개발 환경 현황 (호스트 PC + VirtualBox)」 삭제
- [ ] 호스트에서 `vagrant --version` 확인 (없으면 설치)
- [ ] controlnode RAM 축소: VM을 끈 상태에서 `VBoxManage modifyvm ubuntu-server-01 --memory 2048` 💡
- [ ] VM을 많이 켤 때는 호스트의 로컬 MySQL, Docker, 8080 Java 앱을 종료 (약 3~5GB 확보, 추정)
- [ ] 전용 SSH 키 생성 후 공개키 전달
- [ ] 3단계 전에 GitHub App과 토큰 생성
- [ ] 7단계 이후 백업을 외장 디스크로 정기 복사

## 5. 리스크

| 리스크 | 영향 | 대응 |
|---|---|---|
| PC 한 대에 모든 VM이 있음 | PC가 꺼지면 서비스 중단, 디스크 고장 시 백업까지 유실 | 외장 디스크로 정기 복사, 시연 시간에만 가동 |
| RAM·CPU 부족 | VM 할당 합계가 호스트 RAM에 가까우면 호스트 swap·전체 지연 발생. vCPU 합계(24)가 호스트 16스레드보다 많아 빌드·분석 동시 실행 시 느려짐 | VM 스펙 조정(합계 41GB), 단계별 기동, VM 안 메모리 상한, 호스트 개발 프로세스 종료, 부족 시 ci-01 → worker1 순으로 복구 (「아키텍처 개요」 3장 「자원 운영 방안」) 💡 |
| 호스트가 Wi-Fi에 의존 | 연결이 끊기면 이미지 pull 실패 | 가능하면 유선 사용 |
| Vagrant와 VirtualBox 7.2 호환 | vagrant up 실패 가능 | Vagrant를 최신 버전으로 설치 |
| 무료 GitHub 조직의 브랜치 보호 제한 | main에 직접 push 가능 | 팀 약속과 Jenkinsfile 로직으로 보완 |
| Jenkins가 사설망에 있음 | GitHub Webhook 직접 수신 불가 | 폴링으로 시작, 이후 Funnel 검토 |
| 기술 범위가 큼 | 일정 지연 | 단계별 완료 기준, 보류 항목 명시 |
| 팀 문서와 이 초안의 내용 불일치(팀 문서는 MariaDB·Discord, 초안은 MySQL 8·Slack) | 팀원마다 다른 기준으로 작업 | 회의에서 기준 문서 확정 후 팀 문서 갱신 |
