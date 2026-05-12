---
title: "이미지 변환 / 압축 SaaS"
kind: knowledge
project: project-ideas
agent: engineering-agent/tech-lead
status: idea
created_at: 2026-05-12T23:00:00+09:00
tags:
  - project-idea
  - monetization
  - solo-founder
  - saas-tiny-image-tools
---

# 이미지 변환 / 압축 SaaS

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-12 | engineering-agent/tech-lead | 최초 |

**[[project-ideas|↑ project-ideas/]]**

## 개요

PNG/JPG/WebP 변환 + 압축 + bulk 처리. 단순 도구지만 수억 명 사용자.

## 수익 모델

freemium — 무료 N MB / 월 → Pro 월 $5-15 (TinyPNG: $25/mo). API 사용량 과금.

## 실제 레퍼런스

TinyPNG (~ARR 수백만$) / Squoosh (Google, 오픈소스) / Compressor.io / ezgif (운영자 단독, AdSense + Pro).

## 기술 스택

Cloudflare Workers + WebAssembly (mozjpeg / Squoosh codec) / S3 / Stripe.

## 진입 장벽

낮음 (1-2주 MVP). 차별화는 속도 / batch / API.

## 한국 시장 변형

한글 OG 이미지 자동 생성 + 블로그용 일괄 압축 SaaS. 워드프레스 한국 사용자 타겟.

## 학습 자료

https://github.com/GoogleChromeLabs/squoosh / TinyPNG API docs / Pieter Levels playbook
