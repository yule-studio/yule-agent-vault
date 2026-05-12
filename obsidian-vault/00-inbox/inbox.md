# 00-inbox/ — 분류 전 임시

> 어디에 둘지 모르는 노트의 첫 도착지. 정기적으로 비우고 적절한 폴더로
> 이주시킨다.

**[[../index|↑ vault 인덱스]]**

## 하위 영역

- [[links/|links/]] — 외부 URL 빠른 저장 (나중에 30-resources 로 이주)
- [[quick-notes/|quick-notes/]] — 떠오른 아이디어 / 메모
- [[unsorted/|unsorted/]] — 분류 미정 노트

## 운영 규칙

1. **임시 저장소**. 일주일 이상 inbox 에 남은 노트는 위험 — 분류 미루지
   말 것.
2. **분류 후 이주**:
   - 외부 자료 요약 → `30-resources/<category>/`
   - 개념 정리 → `20-areas/<area>/`
   - 코드 조각 → `50-snippets/<lang>/`
   - 프로젝트 관련 → `10-projects/<project>/`
   - 더 안 볼 노트 → `90-archive/`
3. **메타데이터 약식 OK**. inbox 단계에서는 frontmatter / 컨벤션
   엄격 적용 안 함. 분류 시점에 정식 컨벤션으로 변환.

## 자동화 에이전트 진입

에이전트가 `kind` 를 결정 못 할 때 `00-inbox/unsorted/` 로 라우팅
(`export.py:_yule_vault_kind_to_folder` 의 fallback). 운영자가 inbox
스윕 시점에 적절한 폴더로 이주.

## 관련

- [[../index|↑ vault 인덱스]]
- [[../CLAUDE|운영 규칙]]
