# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal blog (https://Jang-zn.github.io) built with **Jekyll** using the **Chirpy theme** (v5.0.1). The site is in Korean, owned by 장우영 (Jang-zn). Timezone is Asia/Seoul.

## Common Commands

```bash
# Install Ruby dependencies
bundle install

# Run local dev server (http://127.0.0.1:4000, with live reload)
bash tools/run.sh
# or directly:
bundle exec jekyll s -H 0.0.0.0 -l

# Build for production
JEKYLL_ENV=production bundle exec jekyll b

# Test HTML output
bundle exec htmlproofer --disable-external --check-html --allow_hash_href _site

# Build JS assets (requires npm install first)
npx gulp

# Dry-run deploy (build + test, no push)
bash tools/deploy.sh --dry-run
```

## Architecture

- **Theme**: `jekyll-theme-chirpy` (imported as a gem in `Gemfile`, configured in `_config.yml`)
- **Posts**: `_posts/` organized by category subdirectories (일상/, 프로젝트/)
- **Tabs**: `_tabs/` — sidebar navigation pages (about, archives, categories, tags)
- **Layouts**: `_layouts/` — page templates extending the theme
- **Includes**: `_includes/` — reusable HTML partials
- **Data**: `_data/` — site data files (locales, contact info, share config)
- **Sass**: `_sass/` — custom stylesheets
- **JavaScript**: `_javascript/` — JS source organized into commons/, copyright/, utils/; bundled via Gulp (`gulpfile.js/`)
- **Assets**: `assets/` — static files (CSS, JS, images)
- **Tools**: `tools/` — shell scripts for init, build, deploy, and release

## Deployment

Automated via GitHub Actions (`.github/workflows/pages-deploy.yml`). On push to `main`, it builds the site and deploys to the `gh-pages` branch using `tools/deploy.sh`.

## Writing Posts

Posts go in `_posts/<category>/` with the filename format `YYYY-MM-DD-title.md`. Front matter should include `layout: post`, and may include `categories`, `tags`, `toc`, `comments`, etc. Posts default to `layout: post` with comments and TOC enabled.

### 글쓰기 톤

- **무미건조, 시니컬**: 짧고 뚝뚝 끊기는 문장. 감탄사 금지 ("진짜", "미친", "신기한", "깔끔한")
- **반말 구어체**: "~했다", "~인 거다", "말이 안 된다"
- **과잉 표현 금지**: "꽤 의미있는", "진짜 말이 안 되는 속도" 같은 거 쓰지 않는다
- **기술 결정은 건조하게**: 왜 이렇게 했는지 사실만 서술. 감정 과잉 X
- **비유 필수**: 기술 개념을 설명할 때 쉽고 이해하기 쉬운 비유를 들어 설명한다. 일상생활(화장실 문, 택배, 냉장고, 식당 등)에서 가져온 건조한 비유. 비유도 톤에 맞게 시니컬하고 짧게

### 포스트 카테고리

| 카테고리 | 폴더 | 용도 |
|---------|------|------|
| 프로젝트 | `_posts/프로젝트/<프로젝트명>/` | 프로젝트 개발 과정 연재 |
| 학습 | `_posts/학습/` | 기술 개념 학습 + 프로젝트 적용 사례 |

### 프로젝트 포스트 룰

- **연재식**: 날짜순으로 이어지는 시리즈. 이전 포스트 링크 참조 ("저번에 ~했는데")
- **Claude Code 협업 내용 필수**: 어떤 요청을 했고, 뭐가 잘됐고, 뭐가 안 됐는지 구체적으로
- **내 고민/결정 과정 필수**: 왜 이 기술을 선택했는지, 뭘 고려했는지, 뭘 버렸는지
- **커밋 메세지 인용 금지**: 영어 커밋 메세지를 코드블럭으로 인용하지 않는다. 한글로 풀어쓰거나 생략
- **코드 블럭은 핵심만**: 전체 코드 붙여넣기 금지. 의사코드나 핵심 3~5줄만
- **기술 용어**: Port & Adapter, 분산 락, JWT interceptor 같은 기술 용어는 그대로 사용. 단, 일반인이 모를 용어(새니타이즈, 오케스트레이션 등)는 쉽게 풀어쓴다
- **불필요한 기능은 빼기**: 실제로 안 쓸 기능(2FA/TOTP, AI 콘텐츠 심사 등)은 포스트에서도 언급하지 않는다

### 학습 포스트 룰

- **"적용해봤다" 톤**: 교과서가 아니라 "이런 문제가 있어서 이렇게 적용했다" 식으로
- **프로젝트 연결 필수**: 어떤 프로젝트 어떤 기능에 적용했는지 링크 포함
- **작년 학습/멘토링 회고**: "작년에 배운 게 이번에 써먹혔다" 식의 연결. 단, "항해" 같은 특정 과정명은 직접 언급하지 않고 "작년 학습 과정", "멘토링" 등으로 표현
- **포스트 간 상호 링크**: 관련 학습 포스트끼리 서로 참조 (분산 락 ↔ 캐시 스탬피드 ↔ 멱등성)
- **비교 표 활용**: 기술 선택지 비교는 표로 정리

### Front Matter 템플릿

프로젝트 포스트:
```yaml
---
title: "[프로젝트명] 포스트 제목"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: YYYY-MM-DD
categories: [프로젝트, 프로젝트명]
tags: [프로젝트, 프로젝트명, ...]
render_with_liquid: false
---
```

학습 포스트:
```yaml
---
title: "[학습] 포스트 제목"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: YYYY-MM-DD HH:mm:ss +0900
categories: [학습, Backend]
tags: [학습, 키워드1, 키워드2, ...]
render_with_liquid: false
---
```

### 하지 말 것

- 감탄사, 흥분 표현
- 영어 커밋 메세지 인용
- 전체 코드 복붙
- 실제로 안 쓸 기능 언급
- 특정 교육 과정명 직접 언급
- 원본 학습자료 그대로 복사
