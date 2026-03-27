---
title: "[학습] DOM Observer와 뷰포트 스크롤 패턴 — MutationObserver, scrollIntoView"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-27 13:00:00 +0900
categories: [학습, Frontend]
tags: [학습, MutationObserver, ResizeObserver, scrollIntoView, DOM, 코치마크]
difficulty: ⭐5
render_with_liquid: false
---

## "아직 없는 요소"를 기다리는 문제

코치마크 시스템을 만들다 보면 한 가지 난감한 상황이 온다. 타겟 요소가 아직 DOM에 없는 거다.

탭을 전환해야 나타나는 요소, 비동기 로딩으로 늦게 렌더되는 요소, BottomSheet를 열어야 보이는 요소. 이런 것들에 스포트라이트를 비추려면 "나타날 때까지 기다려야" 한다.

기다리는 방법은 두 가지다.

1. **polling** — 5초마다 `document.querySelector`를 호출한다
2. **Observer** — 브라우저한테 "나타나면 알려줘"라고 맡긴다

택배가 올지 안 올지 모르는데 현관 앞을 5초마다 확인하러 나가는 게 polling이다. MutationObserver는 현관 카메라 알림이다. 택배가 도착하면 알림이 온다. 직접 나갈 필요 없다.

## MutationObserver — DOM 변화 감지

MutationObserver(변이 관찰자)는 DOM 트리의 변화를 감시하는 브라우저 내장 API다. 자식 노드가 추가되거나 제거되면 등록한 콜백이 호출된다.

핵심 옵션 두 가지:

| 옵션 | 의미 | 성능 영향 |
|------|------|----------|
| `childList: true` | 직계 자식 추가/제거 감시 | 낮음 |
| `subtree: true` | 하위 전체 감시 | 높음 |

`subtree: true`를 `document.body`에 걸면 전체 DOM을 감시한다. 비용이 크니까 반드시 타임아웃을 걸어야 한다.

```typescript
const mo = new MutationObserver(() => {
  const found = document.querySelector(selector);
  if (found) {
    mo.disconnect(); // 찾았으면 감시 중단
    startObserving(found);
  }
});
mo.observe(document.body, { childList: true, subtree: true });
```

콜백에서 타겟을 찾으면 즉시 `disconnect()`로 감시를 해제한다. 택배 도착 확인했으면 카메라를 끄는 거다.

### 타임아웃 패턴

요소가 영영 안 나타날 수도 있다. 3초 제한을 건다.

```typescript
const timeout = setTimeout(() => {
  mo.disconnect();
  skipStep(); // 이 스텝은 건너뛴다
}, 3_000);
```

요소를 찾으면 `clearTimeout`으로 타이머를 해제하고, 못 찾으면 3초 후 자동으로 다음 스텝으로 넘어간다.

### cleanup — disposed 플래그

MutationObserver 콜백은 마이크로태스크 큐에서 실행된다. `disconnect()` 직후에도 이미 큐에 들어간 콜백이 실행될 수 있다. 문 닫았는데 이미 들어온 사람이 있는 상황이다.

```typescript
let disposed = false;
const mo = new MutationObserver(() => {
  if (disposed) return; // 이미 정리됨
  // ...
});
return () => { disposed = true; mo.disconnect(); };
```

`disposed` 플래그로 "정리 완료 후 콜백"을 차단한다.

## ResizeObserver로 위치 추적 — 이전 포스트와의 차이

[이전 포스트](/posts/SVG-Mask와-ResizeObserver로-코치마크-만들기/)에서 ResizeObserver를 다뤘다. 거기서는 SVG Mask의 스포트라이트 rect 좌표를 계산하는 데 썼다. "구멍의 위치를 실시간으로 맞추는" 용도였다.

이번에는 초점이 다르다. **layout shift 대응**이다.

layout shift(레이아웃 밀림)는 요소의 위치가 예상 밖으로 바뀌는 현상이다. 이미지가 로드되면서 아래 콘텐츠가 밀리거나, 탭 전환으로 높이가 달라지거나. 코치마크 입장에서는 타겟 요소가 갑자기 다른 곳으로 가버리는 거다.

### ResizeObserver + scroll 이벤트 병행

ResizeObserver는 **크기** 변화만 감지한다. 스크롤로 인한 **뷰포트 기준 위치** 변화는 못 잡는다. 그래서 scroll 이벤트를 같이 건다.

| 상황 | ResizeObserver | scroll 이벤트 |
|------|:-:|:-:|
| 이미지 로드로 레이아웃 밀림 | 감지 | 미감지 |
| 사용자 스크롤 | 미감지 | 감지 |
| 윈도우 리사이즈 | 감지 | 미감지 |

둘 다 콜백에서 `getBoundingClientRect()`를 호출해 위치를 다시 측정한다.

### capture: true

scroll 이벤트를 등록할 때 `capture: true`를 붙인다.

```typescript
window.addEventListener('scroll', update, true);
```

왜냐. 스크롤은 중첩 컨테이너에서 발생할 수 있다. `overflow: auto`인 div 안에서 스크롤이 일어나면, 그 이벤트는 window까지 버블링되지 않는다. `capture: true`면 이벤트가 "내려가는" 과정에서 잡으므로 모든 스크롤 컨테이너의 스크롤을 감지한다.

### 중복 업데이트 방지

scroll 이벤트는 자주 발생한다. 매번 Zustand store를 업데이트하면 리렌더가 과도해진다. 이전 위치와 비교해서 변화가 없으면 스킵한다.

```typescript
const prev = prevRectRef.current;
if (prev && prev.left === rect.left && prev.top === rect.top
    && prev.width === rect.width && prev.height === rect.height) {
  return; // 변화 없음, 스킵
}
```

`getBoundingClientRect()`는 매번 새 DOMRect 객체를 반환하므로 `===`로 비교하면 항상 false다. 각 값을 개별 비교해야 한다.

## scrollIntoView — 화면 밖 요소 가져오기

`scrollIntoView()`는 요소가 보이도록 자동 스크롤하는 브라우저 API다. 교과서에서 특정 단어에 형광펜을 칠하려면 먼저 그 페이지를 펼쳐야 한다. scrollIntoView가 "그 페이지 펼치기"다.

### block 옵션 비교

| 옵션 | 동작 | 적합한 상황 |
|------|------|------------|
| `start` | 요소를 뷰포트 상단에 | 문서 상단 이동 |
| `center` | 요소를 뷰포트 중앙에 | 검색 결과 하이라이트 |
| `end` | 요소를 뷰포트 하단에 | 채팅 메시지 |
| `nearest` | 최소한의 스크롤만 | 코치마크 |

코치마크에서는 `nearest`를 쓴다. 이미 보이면 스크롤하지 않고, 화면 밖이면 최소한만 이동한다. 사용자 입장에서 갑자기 화면이 확 움직이면 당황스럽다.

### behavior: 'smooth'의 비동기 특성

`scrollIntoView({ behavior: 'smooth' })`는 비동기다. 호출 즉시 스크롤이 완료되지 않는다. 문제가 될 것 같지만, 이미 scroll 이벤트 리스너가 등록되어 있으므로 스크롤이 진행되는 동안 `update()`가 계속 호출된다. 스포트라이트 위치가 실시간으로 갱신되어 최종 위치에 정확히 안착한다.

## 뷰포트 안에 있는지 판단

scrollIntoView를 무조건 호출하면 `nearest`라도 브라우저가 레이아웃 계산을 수행한다(layout thrashing). 이미 보이는 요소에 대해 불필요한 비용이 발생한다.

### guard 패턴

```typescript
const rect = el.getBoundingClientRect();
const isOutOfView = rect.top < 0 || rect.bottom > window.innerHeight;
if (isOutOfView) {
  el.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
}
```

`getBoundingClientRect()`로 뷰포트 기준 위치를 측정하고, 밖에 있을 때만 scrollIntoView를 호출한다.

판단 기준 두 가지:

| 판단 | 조건 | 용도 |
|------|------|------|
| 완전히 보임 | `top >= 0 && bottom <= innerHeight` | 스크롤 불필요 |
| 일부라도 보임 | `top < innerHeight && bottom > 0` | 부분 노출 허용 시 |
| 완전히 안 보임 | `top > innerHeight \|\| bottom < 0` | 반드시 스크롤 |

코치마크에서는 "요소의 상단이나 하단이 잘렸으면 스크롤"로 판단한다.

## 코치마크에서의 조합

세 Observer/API가 하나의 흐름으로 엮인다.

```
step 변경
  → document.querySelector로 타겟 찾기
  → 없으면: MutationObserver로 대기 (3초 타임아웃)
  → 찾으면: 뷰포트 밖인지 확인
    → 밖이면: scrollIntoView로 이동
    → 안이면: 바로 추적 시작
  → ResizeObserver + scroll 이벤트로 위치 실시간 추적
  → getBoundingClientRect() → prevRect 비교 → store 업데이트
  → SpotlightSvg 렌더링
```

각각의 역할을 정리하면:

| API | 역할 | 감시 대상 |
|-----|------|----------|
| MutationObserver | 아직 없는 요소 대기 | DOM 트리 변화 |
| scrollIntoView | 화면 밖 요소를 뷰포트 안으로 | - |
| ResizeObserver | 크기 변화에 따른 위치 재계산 | 요소 크기 |
| scroll event (capture) | 스크롤에 따른 위치 재계산 | 모든 스크롤 컨테이너 |

하나만으로는 안 된다. MutationObserver만 있으면 찾은 요소가 화면 밖일 수 있고, scrollIntoView만 있으면 이후 layout shift에 대응 못 한다. ResizeObserver만 있으면 스크롤 위치 변화를 놓친다. 세 개를 조합해야 "요소 등장 감지 → 뷰포트 이동 → 실시간 추적"이 완성된다.

상세한 SVG Mask 스포트라이트 구현은 [이전 포스트](/posts/SVG-Mask와-ResizeObserver로-코치마크-만들기/), 코치마크 확장 과정은 [코치마크 프로젝트 포스트](/posts/코치마크-확장과-환경-분리/)에서 다뤘다.

## 정리

| 개념 | 핵심 | 주의사항 |
|------|------|---------|
| MutationObserver | DOM 변화 감지, 요소 대기 | `subtree: true`는 비용 큼, 타임아웃 필수 |
| ResizeObserver | 크기 변화 감지 | 스크롤 위치는 못 잡음, scroll 이벤트 병행 |
| scrollIntoView | 요소를 뷰포트로 이동 | guard 패턴으로 불필요한 호출 방지 |
| capture: true | 중첩 스크롤 컨테이너 대응 | 버블링 안 되는 scroll 이벤트 잡기 |
| disposed 플래그 | 비동기 콜백 안전 차단 | 마이크로태스크 큐 방어 |
| prevRect 비교 | 중복 store 업데이트 방지 | DOMRect는 매번 새 객체, 값 비교 필요 |
