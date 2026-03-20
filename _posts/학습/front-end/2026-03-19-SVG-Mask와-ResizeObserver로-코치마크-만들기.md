---
title: "[학습] SVG Mask와 ResizeObserver — 코치마크 스포트라이트 구현"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-19 20:00:00 +0900
categories: [학습, Frontend]
tags: [학습, SVG, ResizeObserver, 코치마크, React]
render_with_liquid: false
---

## 코치마크를 만들어야 했다

[TasteNote](/posts/camelCase-마이그레이션/)에 튜토리얼 오버레이를 넣었다. "이 버튼을 눌러보세요" 같은 가이드. 특정 요소만 밝게 보이고 나머지는 어둡게 — 스포트라이트 효과가 필요하다.

## SVG Mask로 구멍 뚫기

스포트라이트의 원리는 단순하다. 화면 전체를 어둡게 덮고, 타겟 요소 부분만 뚫는 것. SVG `<mask>`가 이걸 해준다.

mask의 규칙: **흰색 = 보임, 검정색 = 숨김**.

| 영역 | mask 색상 | 결과 |
|------|-----------|------|
| 전체 화면 | 흰색 | 반투명 오버레이가 보임 (어두움) |
| 타겟 요소 | 검정색 | 반투명 오버레이가 숨겨짐 (밝음) |

"어둡게 칠한 사각형에서 타겟 부분만 지우개로 지운다"는 발상이다.

```tsx
<svg style={{ position: 'fixed', inset: 0 }}>
  <defs>
    <mask id="spotlight-mask">
      <rect width="100%" height="100%" fill="#fff" />
      <rect x={left} y={top} width={w} height={h}
            rx={12} fill="#000" />
    </mask>
  </defs>
  <rect width="100%" height="100%"
        fill="rgba(0,0,0,0.55)"
        mask="url(#spotlight-mask)" />
</svg>
```

흰색 rect로 전체를 채우고, 검정 rect로 타겟 영역을 덮어쓴다. 그 mask를 반투명 오버레이에 적용하면 타겟 부분만 투명해진다. `rx={12}`는 SVG의 `border-radius` — 둥근 모서리 구멍.

padding도 넣었다. 타겟과 스포트라이트가 딱 붙으면 답답하니까 8px씩 여유를 줬다.

### 왜 CSS가 아닌 SVG Mask인가

| 방법 | 장점 | 단점 |
|------|------|------|
| box-shadow | 간단 | 둥근 구멍 불가, 클릭 투과 어려움 |
| clip-path | CSS만으로 가능 | 복잡한 형태 어려움 |
| canvas | 자유도 높음 | 무겁고 접근성 불리 |
| SVG Mask | 둥근 모서리, 클릭 이벤트, 성능 양호 | SVG 문법 학습 필요 |

SVG Mask를 고른 이유는 `rx`로 둥근 구멍, `onClick` 직접 부착, GPU 합성 레이어에서 처리되는 성능까지 다 충족해서다.

## ResizeObserver로 위치 추적

스포트라이트를 그리려면 타겟 요소의 위치와 크기를 알아야 한다. `getBoundingClientRect()`로 한 번 읽으면 끝일 것 같지만, 그게 아니다. 이미지가 로드되면 레이아웃이 밀리고, 스크롤하면 위치가 바뀌고, 창 크기를 조절하면 또 바뀐다.

`getBoundingClientRect()`는 호출 시점의 스냅샷만 준다. 사진 한 장 찍은 거다. 대상이 움직이면 사진은 소용없다. 실시간 CCTV가 필요하다. `ResizeObserver`가 그 역할이다.

```tsx
useEffect(() => {
  const el = document.querySelector(`[data-coach="${target}"]`);
  if (!el) return;

  const sync = () => updateRect(el.getBoundingClientRect());

  const ro = new ResizeObserver(sync);
  ro.observe(el);

  window.addEventListener('scroll', sync, { passive: true });

  return () => {
    ro.disconnect();
    window.removeEventListener('scroll', sync);
  };
}, [target]);
```

### ResizeObserver + scroll 이벤트 병행

둘 다 필요하다.

| 상황 | ResizeObserver | scroll 이벤트 |
|------|:-:|:-:|
| 이미지 로드로 레이아웃 밀림 | 감지 | 미감지 |
| 사용자 스크롤 | 미감지 | 감지 |
| 윈도우 리사이즈 | 감지 | 미감지 |

ResizeObserver는 크기 변화만 잡는다. 스크롤로 인한 위치 변화는 못 잡는다. 그래서 scroll 이벤트를 같이 건다.

### passive 이벤트 리스너

`{ passive: true }`는 "이 리스너는 `preventDefault()`를 호출하지 않겠다"는 약속이다. 고속도로 톨게이트에서 "하이패스 전용"이라고 표시하면 차가 멈추지 않고 통과하는 것과 같다. 브라우저가 이 약속을 믿고 리스너 실행을 기다리지 않고 스크롤을 바로 실행한다. 스크롤이 버벅이지 않는다.

### cleanup

`disconnect()`를 빼먹으면 이전 요소에 대한 참조가 남아서 메모리 누수가 생기고, 이전 요소의 변화가 현재 스포트라이트를 움직인다. 코치마크 스텝이 바뀔 때마다 이전 observer를 해제하고 새 요소를 감시해야 한다.

### 브라우저 Observer 4종

| Observer | 감시 대상 | 대표 사용처 |
|----------|-----------|-------------|
| ResizeObserver | 요소 크기 변화 | 반응형 차트, 스포트라이트 |
| IntersectionObserver | 뷰포트 교차 | 무한 스크롤, lazy loading |
| MutationObserver | DOM 트리 변화 | 동적 콘텐츠 감지 |
| PerformanceObserver | 성능 메트릭 | Core Web Vitals 측정 |

이번에 ResizeObserver를 처음 써봤는데, IntersectionObserver만 알고 있던 것에 비하면 확실히 용도가 다르다. "크기 변화 → 위치 재계산"이 필요한 경우에는 ResizeObserver가 맞다.

## 정리

SVG Mask로 구멍을 뚫고, ResizeObserver로 구멍의 위치를 추적한다. 이 두 개가 코치마크 스포트라이트의 전부다. 상태 관리와 완료 저장은 [Zustand와 localStorage 팩토리](/posts/Zustand-스토어와-localStorage-팩토리/)에서 다룬다.
