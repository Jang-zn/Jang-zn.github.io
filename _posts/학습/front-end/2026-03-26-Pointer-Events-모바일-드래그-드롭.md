---
title: "[학습] Pointer Events API — 모바일 드래그&드롭 직접 구현"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-26 23:00:00 +0900
categories: [학습, Frontend]
tags: [학습, React, Pointer Events, 드래그드롭, 모바일, 터치]
render_with_liquid: false
---

## HTML5 DnD API는 모바일에서 안 된다

[TasteNote 요리모드](/posts/요리모드-v2-구현과-UX-개선/)에서 조리 순서 드래그&드롭을 구현했다. HTML5의 기본 DnD API(`draggable`, `ondragstart`)는 데스크톱에서만 된다. 터치 이벤트를 지원하지 않는다. 토스 앱은 모바일이니까 Pointer Events API를 써야 한다.

라이브러리(`react-beautiful-dnd`, `dnd-kit`)도 있지만 쓰지 않았다. 단순한 세로 목록 재정렬에 라이브러리를 추가하면 번들 크기가 괜히 늘어난다.

## Pointer Events API

마우스, 터치, 스타일러스를 **하나의 인터페이스**로 통합한 API. 이벤트 3개로 드래그 전체를 처리한다.

| 이벤트 | 타이밍 |
|--------|--------|
| `pointerdown` | 손가락을 댔을 때 |
| `pointermove` | 손가락이 움직일 때 |
| `pointerup` | 손가락을 뗐을 때 |

`e.pointerId`로 멀티터치 시 특정 손가락을 추적할 수 있다. 두 손가락이 동시에 화면에 있어도 드래그 중인 손가락만 처리한다.

## 구현 패턴

```typescript
const [dragFrom, setDragFrom] = useState<number | null>(null);
const [pointerId, setPointerId] = useState<number | null>(null);

// 드래그 핸들에서
const handlePointerDown = (idx: number) => (e: React.PointerEvent) => {
  setDragFrom(idx);
  setPointerId(e.pointerId);
};

// window에 이벤트 등록 (pointerdown 이후에만)
useEffect(() => {
  if (dragFrom === null) return;

  const handleMove = (e: PointerEvent) => {
    if (e.pointerId !== pointerId) return; // 다른 손가락 무시
    // 드롭 위치 계산
  };

  const handleUp = (e: PointerEvent) => {
    if (e.pointerId !== pointerId) return;
    // 순서 변경 처리
    setDragFrom(null);
    setPointerId(null);
  };

  window.addEventListener('pointermove', handleMove);
  window.addEventListener('pointerup', handleUp);
  return () => {
    window.removeEventListener('pointermove', handleMove);
    window.removeEventListener('pointerup', handleUp);
  };
}, [dragFrom, pointerId]);
```

`window`에 이벤트를 등록하는 이유: `pointermove`를 드래그 핸들 컴포넌트에만 등록하면, 손가락이 핸들 영역을 벗어나면 이벤트를 못 받는다. 창문을 열어놓고 온도를 측정하는 게 아니라, 방 전체에서 측정하는 거다.

`dragFrom === null`일 때는 리스너를 등록하지 않는다. 불필요한 `pointermove`가 모든 터치마다 발생하면 낭비다.

## 필수 CSS

```css
/* 드래그 핸들에 적용 */
.drag-handle {
  touch-action: none;   /* 브라우저 기본 스크롤 비활성화 */
  user-select: none;    /* 텍스트 선택 방지 */
}
```

`touch-action: none`이 없으면 `pointermove` 중에 브라우저가 스크롤을 같이 처리하려고 한다. 드래그와 스크롤이 동시에 발생해서 UX가 망가진다.

## 드롭 위치 계산

```typescript
const rowRefs = useRef<(HTMLDivElement | null)[]>([]);

const getDropIndex = (clientY: number): number => {
  for (let i = 0; i < rowRefs.current.length; i++) {
    const el = rowRefs.current[i];
    if (!el) continue;
    const { top, height } = el.getBoundingClientRect();
    if (clientY < top + height / 2) return i; // 행의 중간선 위 → 이 위치에 삽입
  }
  return rowRefs.current.length; // 맨 아래
};
```

각 행의 DOM 위치를 측정해서 현재 Y 좌표가 어느 행의 중간선을 넘었는지 판단한다.

## 순서 변경 알고리즘

```typescript
const moveStep = (fromIdx: number, toIdx: number) => {
  const newOrder = [...stepOrder];
  const [moved] = newOrder.splice(fromIdx, 1); // 꺼내고
  newOrder.splice(toIdx, 0, moved);             // 끼워넣기
  setStepOrder(newOrder);
};
```

swap이 아니라 insert 방식이다. swap하면 두 항목이 자리를 바꾸는데, 이건 "끌어서 원하는 위치에 넣기"와 다르다. 책꽂이에서 책을 꺼내서 원하는 자리에 꽂는 거다. 뽑은 자리의 책들이 한 칸씩 당겨지고, 꽂은 자리부터 한 칸씩 밀린다.
