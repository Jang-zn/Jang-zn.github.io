---
title: "[학습] Feature 모듈 설계와 Barrel Export"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-25 21:00:00 +0900
categories: [학습, Frontend]
tags: [학습, React, FSD, Feature, Barrel Export, 아키텍처]
difficulty: ⭐4
render_with_liquid: false
---

## 기능이 늘어나면 파일도 늘어난다

[TasteNote에 인앱결제 모듈](/posts/토스트-통일과-인앱결제-UI/)을 추가하면서 `features/purchase/`를 만들었다. 기능 단위로 파일을 묶는 Feature 모듈 패턴과 Barrel Export를 정리한다.

## Feature-Sliced Design (FSD)

TasteNote의 레이어 구조:

```
pages/        → 페이지 컴포넌트
features/     → 기능 모듈 (비즈니스 로직 + UI)
entities/     → 도메인 엔티티 (타입, 모델)
shared/       → 공유 유틸, UI 컴포넌트
```

의존 방향: `pages → features → entities → shared`. 위에서 아래로만 참조한다. features끼리 직접 참조하지 않는다. 아파트 층수와 같다. 3층이 1층 편의점을 이용할 수 있지만, 3층이 4층 빨래방을 이용하려면 엘리베이터(shared)를 거쳐야 한다.

## Feature 모듈 내부 구조

```
features/purchase/
├── api/          → API 호출 함수
├── lib/          → 비즈니스 로직 훅
│   ├── useAdFreeStatus.ts
│   ├── useGrantPurchase.ts
│   ├── useRequestRefund.ts
│   └── usePurchaseFlow.ts
├── ui/           → UI 컴포넌트
│   └── PolicyBottomSheet.tsx
└── index.ts      → Barrel export
```

`api/`는 서버 통신, `lib/`은 로직, `ui/`는 화면. 이 구조가 반복된다. 새 기능을 추가할 때 "어디에 뭘 넣지?"를 고민할 필요가 없다.

## Barrel Export

`index.ts`에서 외부에 노출할 것만 export한다.

```typescript
// features/purchase/index.ts
export { useAdFreeStatus } from './lib/useAdFreeStatus';
export { usePurchaseFlow } from './lib/usePurchaseFlow';
export { PolicyBottomSheet } from './ui/PolicyBottomSheet';
```

페이지에서는 이렇게 import한다:

```typescript
import { useAdFreeStatus, PolicyBottomSheet } from '@/features/purchase';
```

내부 구현(`api/`, 개별 파일 경로)이 감춰진다. 호텔 프론트 데스크와 같다. 투숙객은 프론트에서 필요한 서비스를 요청하지, 직접 주방이나 세탁실을 찾아가지 않는다.

### Barrel Export의 장점

1. **캡슐화**: 내부 구조가 바뀌어도 index.ts만 수정하면 됨
2. **명시적 공개 API**: export된 것만 외부에서 쓸 수 있음
3. **import 경로 단순화**: 깊은 경로 대신 모듈명으로

### 주의점

트리셰이킹이 안 되는 번들러에서는 barrel export가 불필요한 코드를 포함시킬 수 있다. Vite(Rollup)는 트리셰이킹을 잘 하니까 문제없다.

## 프로젝트 적용

기존 feature 모듈들도 같은 구조:

```
features/recipe/      → 레시피 CRUD
features/parse-job/   → URL 파싱
features/ad/          → 광고
features/purchase/    → 인앱결제 (신규)
```

새 모듈을 만들 때 디렉토리 구조를 복사하고 내용만 채우면 된다. [BE의 멀티모듈 구조](/posts/멀티모듈-백엔드-세팅/)와 같은 원리다. 모듈 경계가 명확하면 팀이 커져도 충돌이 줄어든다.
