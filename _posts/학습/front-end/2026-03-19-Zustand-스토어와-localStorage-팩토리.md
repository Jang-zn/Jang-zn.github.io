---
title: "[학습] Zustand 스토어와 localStorage 팩토리 — 코치마크 상태 관리"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-19 20:30:00 +0900
categories: [학습, Frontend]
tags: [학습, Zustand, localStorage, 상태관리, React]
render_with_liquid: false
---

## 코치마크에 상태가 필요하다

[SVG Mask와 ResizeObserver](/posts/SVG-Mask와-ResizeObserver로-코치마크-만들기/)로 스포트라이트를 그렸다. 근데 "지금 몇 번째 스텝인지", "어떤 요소를 가리키는지", "완료했는지"를 관리해야 한다. Zustand 스토어와 localStorage 팩토리로 해결했다.

## Zustand 스토어 — 상태 머신으로 설계

코치마크 스토어는 사실상 상태 머신이다. Idle → Active → Done.

```
[Idle] --start()--> [Active] --nextStep()/updateRect()--> [Active]
                    [Active] --complete()/skip()--> [Done] --> [Idle]
```

```tsx
interface CoachMarkState {
  // 상태
  tutorialId: string | null;
  steps: CoachStep[];
  currentIdx: number;
  rect: DOMRect | null;

  // 액션
  start: (id: string, steps: CoachStep[]) => void;
  nextStep: () => void;
  updateRect: (r: DOMRect) => void;
  skip: () => void;
  complete: () => void;
}
```

Zustand의 특징은 상태와 액션을 한 인터페이스에 선언한다는 것. Redux처럼 action type 상수, action creator, reducer를 분리하지 않는다. `create((set, get) => ({...}))` 한 줄로 스토어가 만들어진다.

### set과 get

`set()`으로 상태를 변경하고, `get()`으로 액션 내부에서 현재 상태를 참조한다.

```tsx
nextStep: () => {
  const { currentIdx, steps } = get();
  if (currentIdx < steps.length - 1) {
    set({ currentIdx: currentIdx + 1, rect: null });
  }
},
```

`nextStep`이 호출되면 현재 인덱스를 읽고, 마지막 스텝이 아니면 다음으로 넘긴다. `rect: null`로 초기화하는 건 새 스텝의 타겟 요소 위치를 아직 모르기 때문이다. ResizeObserver가 잡아주면 `updateRect`로 채워진다.

### 셀렉터 패턴 — 리렌더 최소화

```tsx
// 스토어 전체를 구독하면 어떤 값이든 바뀔 때 리렌더링
const store = useCoachMarkStore(); // ← 비효율

// 필요한 값만 구독
const rect = useCoachMarkStore((s) => s.rect); // ← rect 바뀔 때만 리렌더
```

`SpotlightSvg`는 `rect`만 구독하고, `Tooltip`은 `currentIdx`와 `steps`를 구독한다. 스크롤할 때마다 `rect`가 바뀌는데, 그때 Tooltip은 리렌더되지 않는다.

### 컴포넌트 밖에서 접근

Zustand 스토어는 React 컴포넌트 밖에서도 접근 가능하다. `useStore.getState()`로.

```tsx
useCoachMarkStore.getState().start('home', homeSteps);
```

이 특성이 localStorage 팩토리와 연결된다. `complete()` 액션 내부에서 localStorage에 완료 플래그를 저장해야 하는데, 커스텀 훅은 컴포넌트 밖에서 못 쓴다. 팩토리 함수는 쓸 수 있다.

## localStorage 팩토리 패턴

localStorage를 직접 쓰면 키 이름 오타, try-catch 중복, 값 포맷 불일치 같은 문제가 생긴다. 팩토리 패턴으로 공통 로직을 캡슐화했다.

```tsx
// 영구 플래그
export function createFlag(prefix: string) {
  return {
    isDone: (id: string): boolean => {
      try { return localStorage.getItem(prefix + id) === 'true'; }
      catch { return false; }
    },
    markDone: (id: string): void => {
      try { localStorage.setItem(prefix + id, 'true'); }
      catch { /* quota exceeded */ }
    },
  };
}

// 일간 플래그 — 오늘 하루만 유효
export function createDailyFlag(prefix: string) {
  const today = () => new Date().toISOString().slice(0, 10);
  return {
    isDismissedToday: (id: string): boolean => {
      try { return localStorage.getItem(prefix + id) === today(); }
      catch { return false; }
    },
    dismiss: (id: string): void => {
      try { localStorage.setItem(prefix + id, today()); }
      catch { /* quota exceeded */ }
    },
  };
}
```

사용하는 쪽에서는 prefix만 넣으면 된다.

```tsx
const coachStorage = createFlag('coach_done_');
coachStorage.isDone('home');     // localStorage.getItem('coach_done_home') === 'true'
coachStorage.markDone('home');   // localStorage.setItem('coach_done_home', 'true')

const bannerFlag = createDailyFlag('banner_dismissed_');
bannerFlag.isDismissedToday('promo'); // 오늘 날짜와 비교
bannerFlag.dismiss('promo');          // 오늘 날짜 저장, 내일이면 자동 만료
```

### QuotaExceeded 처리

localStorage는 약 5MB 제한이 있다. 꽉 차면 `setItem`에서 에러가 난다. 팩토리 안에서 try-catch로 조용히 실패시킨다. 코치마크의 경우 저장 실패하면 다음에 앱 열 때 튜토리얼이 다시 나올 뿐이다. 앱이 죽는 것보다 낫다.

### 왜 클래스나 훅이 아닌 팩토리인가

| 방식 | 장점 | 단점 |
|------|------|------|
| 팩토리 함수 | 가볍고, React 밖에서 사용 가능 | 상속/확장 어려움 |
| 클래스 | 타입 체계적 | React에서 어색 |
| 커스텀 훅 | React 라이프사이클 연동 | 컴포넌트 안에서만 사용 가능 |

팩토리를 고른 이유는 Zustand 스토어의 `complete()` 액션에서 호출해야 하기 때문이다. 훅은 컴포넌트 밖에서 못 쓴다.

## Zustand vs 다른 상태 관리

| 기준 | Zustand | Redux Toolkit | Jotai | React Context |
|------|---------|---------------|-------|---------------|
| 보일러플레이트 | 매우 적음 | 중간 | 적음 | 적음 |
| 번들 크기 | ~1KB | ~11KB | ~3KB | 0 |
| 컴포넌트 외부 접근 | getState | 설정 필요 | 불가 | 불가 |
| 셀렉터 리렌더 제어 | 기본 제공 | useSelector | atom 단위 | 전체 리렌더 |

코치마크처럼 "여러 컴포넌트가 공유하는 UI 상태 + 컴포넌트 밖에서도 접근"이 필요한 경우에 Zustand가 딱이다. [TanStack Query](/posts/TanStack-Query-캐시-키-설계/)는 서버 상태 관리용이고, Zustand는 클라이언트 상태 관리용. 용도가 다르다.

## 정리

코치마크의 상태 흐름: Zustand 스토어가 현재 스텝과 위치를 관리하고, 완료 시 localStorage 팩토리로 플래그를 저장한다. 다음에 앱을 열면 플래그를 확인해서 이미 완료한 튜토리얼은 건너뛴다.

팩토리 패턴은 BE의 [캐시 키 설계](/posts/Redis-캐시-직렬화-충돌/)랑 비슷한 발상이다. prefix로 네임스페이스를 분리하고, 공통 로직(try-catch, 포맷)을 한 곳에 모은다. "같은 패턴이 3번 반복되면 추출한다"는 원칙.
