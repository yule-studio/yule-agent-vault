---
title: "가격 추적기 / 재고 알림"
kind: knowledge
project: project-ideas
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T23:00:00+09:00
tags:
  - project-idea
  - monetization
  - solo-founder
  - saas-scraping-price-tracker
---

# 가격 추적기 / 재고 알림

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[project-ideas|↑ project-ideas/]]**

## 개요

상품 URL → 가격 변동 / 재고 발생 시 알림. 쇼핑 시즌 폭발.

## 수익 모델

freemium — Pro $4-19/mo (트래커 수 / 빈도).

## 실제 레퍼런스

Keepa (Amazon, ARR $5M+) / camelcamelcamel / Hotstock (PS5/GPU 추적, exited).

## 기술 스택

Puppeteer / Playwright + Cloudflare Workers + SendGrid.

## 진입 장벽

낮음 (스크래핑은 쉬움). 차별화 = anti-bot 회피 + 카테고리 특화.

## 한국 시장 변형

쿠팡 / 11번가 / 네이버 쇼핑 가격 추적 + 카카오톡 알림. 한정판 / 신상품 (Nike / Adidas).

## 학습 자료

Keepa 운영 인터뷰 / Playwright docs / 카카오톡 메시지 API
