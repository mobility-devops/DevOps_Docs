# runbooks

장애 대응·운영 절차. Alertmanager 알림마다 대응 문서를 하나씩 둔다(아키텍처 8장).

- 파일 이름은 알림 이름을 따른다. 예: `node-down.md`, `laptop-link-down.md`
- 알림의 `runbook_url`에 이 문서 링크를 넣는다.

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
