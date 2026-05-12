# 90-archive/ — 보관

> 더 이상 활성 노트가 아니지만 **삭제하지 않고 보존** 해야 하는 노트.
> 결정 / 리서치 / 작업 로그의 히스토리를 유지하기 위함.

**[[../index|↑ vault 인덱스]]**

## 카테고리

| 카테고리 | 진입점 | 무엇이 들어가는가 |
| --- | --- | --- |
| deprecated-notes | [[deprecated-notes/]] | 더 이상 유효하지 않은 개념 / 패턴 / 트러블 노트 |
| migrated | [[migrated/]] | 다른 vault 또는 외부 시스템으로 옮긴 노트 (참조 유지) |
| old-projects | [[old-projects/]] | 종료된 프로젝트의 `10-projects/<name>/` 전체 |

## 노트 이주 규칙

### 무엇을 archive 하나

1. **deprecated** — 개념 / 패턴 / 라이브러리가 더 이상 권장되지 않음.
   본문은 그대로 두고 frontmatter `status: deprecated` + 첫 줄에
   "Deprecated since YYYY-MM-DD: 후속 [[link]]".
2. **migrated** — 같은 정보가 다른 곳 (외부 wiki / 새 vault) 으로
   이주. 본문은 stub 으로, 첫 줄에 "Moved to <URL or [[link]]>".
3. **old-projects** — 종료된 프로젝트. 폴더 단위로 이주, 내부 구조는
   유지.

### 무엇을 삭제하나

거의 없음. archive 는 보존이 기본 — 삭제는 PII 노출 / 보안 사유 등
예외 케이스만.

## 운영 규칙

1. **archive 도 검색 대상**. Obsidian 그래프에서 회색 처리하더라도
   삭제는 아님.
2. archive 한 노트의 원래 위치에는 **[[link]] stub** 을 남기지 않는다 —
   `90-archive/...` 로 옮기면 그 자체가 신호.
3. 다른 활성 노트가 archive 된 노트를 참조하면 archive 의 첫 줄 후속
   링크를 따라가도록.
4. archive 후 1 년 이상 reference 없으면 다음 리뷰 사이클에서 삭제
   검토.

## 자동화 에이전트

에이전트는 archive 폴더에 직접 쓰지 않는다. 이주는 운영자 결정 +
수동 또는 `yule obsidian` 의 archive 서브명령 (예정).

## 관련

- [[../index|↑ vault 인덱스]]
