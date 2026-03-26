---
title: "[학습] Zustand persist와 세션 스토어 설계"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-26 11:00:00 +0900
categories: [학습, Frontend]
tags: [학습, React, Zustand, persist, localStorage, 상태관리]
difficulty: ⭐5
render_with_liquid: false
---

## 앱을 껐다 켜도 요리 중이었으면 이어가야 한다

TasteNote 요리 모드에서 사용자가 요리 중에 앱을 닫았다가 다시 열면 진행 상태가 날아가면 안 된다. 양파를 볶고 있었는데 앱을 잠깐 닫았다가 열었더니 처음부터 시작하라고 하면 화난다.

Zustand의 `persist` 미들웨어로 세션 상태를 localStorage에 저장한다.

## persist 미들웨어

```typescript
const useCookingSessionStore = create(
  persist(
    (set, get) => ({
      recipeId: null,
      currentStep: 0,
      ingredientActions: {},
      startedAt: null,
      // ...
    }),
    {
      name: 'cooking-session',
    }
  )
);
```

상태가 변경될 때마다 자동으로 localStorage에 직렬화해서 저장한다. 앱을 다시 열면 localStorage에서 복원.

## TTL 패턴 — 24시간 만료

어제 시작한 요리 세션이 오늘도 남아있으면 의미가 없다. 24시간이 지난 세션은 자동 폐기.

```typescript
const isExpired = (session) => {
  if (!session.startedAt) return true;
  const elapsed = Date.now() - session.startedAt;
  return elapsed > 24 * 60 * 60 * 1000;
};
```

스토어 초기화 시 TTL 체크. 만료됐으면 초기 상태로 리셋. 냉장고에 유통기한 지난 음식이 있으면 열 때 자동으로 버리는 거다.

## useShallow로 불필요한 리렌더 방지

Zustand에서 여러 값을 선택하면 기본적으로 매번 새 객체가 만들어져서 리렌더가 발생한다.

```typescript
// Bad: 매번 새 객체 → 매번 리렌더
const { currentStep, ingredientActions } = useCookingSessionStore(state => ({
  currentStep: state.currentStep,
  ingredientActions: state.ingredientActions,
}));

// Good: useShallow로 얕은 비교
const { currentStep, ingredientActions } = useCookingSessionStore(
  useShallow(state => ({
    currentStep: state.currentStep,
    ingredientActions: state.ingredientActions,
  }))
);
```

`useShallow`는 객체의 각 값을 얕은 비교한다. 실제로 변경된 값이 없으면 리렌더하지 않는다. 택배 상자를 열어보고 내용물이 같으면 "새 택배 왔다"고 알리지 않는 거다.

## 파생 상태는 저장하지 않는다

```typescript
// Bad: hasModifications를 별도 상태로 저장
{ hasModifications: false, ingredientActions: {} }

// Good: 기존 데이터에서 계산
const hasModifications = useMemo(() => {
  return Object.values(ingredientActions).some(a => a !== 'USED');
}, [ingredientActions]);
```

파생 상태(derived state)를 store에 저장하면 원본 데이터와 싱크가 안 맞을 위험이 있다. 달력에 "오늘은 목요일"이라고 적어두면 내일은 틀린 정보가 된다. 날짜를 보고 매번 계산하면 항상 맞다.

## 프로젝트 적용

`cookingSessionStore`에 적용:
- persist: localStorage 저장/복원
- TTL: 24시간 만료
- useShallow: CookingPage 컴포넌트에서 6개 값 선택 시 적용
- 파생 상태: `hasModifications`, `completedStepCount`는 `useMemo`로 계산
