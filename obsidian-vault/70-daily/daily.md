# 70-daily/ — 일일 작업 기록

> 하루 단위 짧은 work-log. 어디에 분류할지 모를 때 일단 오늘 노트에
> 두고, 정리할 때 적절한 폴더로 분산.

**[[../index|↑ vault 인덱스]]**

## 구조

```
70-daily/
├── README.md       ← 본 문서
├── templates/      ← 일일 노트 템플릿
└── <YYYY>/         ← 연도별
    ├── <YYYY-MM-DD>.md
    └── ...
```

## 파일명

```
<YYYY-MM-DD>.md
```

예: `2026-05-12.md`. 일일 노트는 예외적으로 **날짜만** 으로 충분 (다른
컨벤션의 `<kind>-<topic>` 형식과 다름).

## 노트 표준 구조

```markdown
# 2026-05-12 (월)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | <역할> | 최초 |

## 한 줄
오늘의 한 줄.

## 작업
- 09:00 — ...
- 14:00 — ...

## 결정 / 발견
정리해야 할 결정·발견 (나중에 `decisions/` / `research/` 로 이주).

## 막힘 / 트러블
정리해야 할 트러블 (나중에 `60-troubleshooting/` 로 이주).

## 내일 / 다음
- TODO

## 관련
- [[../10-projects/.../<오늘 작업 프로젝트>]]
```

## 운영 규칙

1. **일주일 이상 묵힌 daily 는 분류** — 결정 / 트러블 / 학습은 적절한
   폴더로 추출.
2. 추출 후 daily 노트에는 [[link]] 만 남긴다 (히스토리 보존).
3. 매년 새 디렉터리 (`70-daily/2027/`).
4. 템플릿은 `70-daily/templates/` 에 (Obsidian Templater 호환).

## 자동화 에이전트 / scheduled-automation

`yule daily` CLI 가 plan_today / briefing 산출물을 본 폴더에 작성할 수
있음 (옵션). 매핑: yule-studio-agent repo 의 `cli/daily.py`.

## 관련

- [[../index|↑ vault 인덱스]]
- [[../80-templates/README|80-templates]] — 다른 종류 템플릿
