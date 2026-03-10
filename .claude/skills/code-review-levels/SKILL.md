---
name: code-review-levels
description: QA 테스트 코드 심층 리뷰 (Level 3) 및 테스트 아키텍처 리뷰 (Level 4). Playwright + TypeScript 특화.
allowed-tools: Read
---

# Code Review Levels Reference (QA 특화)

`code-reviewer` 에이전트가 Level 3/4 리뷰 시 참조하는 상세 가이드입니다.
Playwright + TypeScript 테스트 코드에 특화된 리뷰 프로토콜을 제공합니다.

## Available References

| Level | File | 용도 |
|-------|------|------|
| Level 3 (Deep) | [deep-review.md](deep-review.md) | 테스트 흐름, Flaky 테스트 탐지, 데이터 격리, 커버리지, 보안 심층 분석 |
| Level 4 (Architecture) | [architecture-review.md](architecture-review.md) | 레이어 준수, POM 패턴, 설정/환경 분리, CI/CD 통합, 재사용성 |

## Usage

`code-reviewer` 에이전트가 Level 3 또는 4 리뷰를 수행할 때 해당 파일을 Read하여 프로토콜을 따릅니다.
