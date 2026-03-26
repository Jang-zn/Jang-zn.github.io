---
title: "[학습] Wake Lock API — 요리 중 화면 꺼짐 방지"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-26 12:00:00 +0900
categories: [학습, Frontend]
tags: [학습, Wake Lock, 모바일, UX, Web API]
difficulty: ⭐4
render_with_liquid: false
---

## 손에 밀가루가 묻어있는데 화면이 꺼진다

TasteNote 요리 모드에서 레시피를 보면서 요리한다. 가만히 놔두면 화면이 자동으로 꺼진다. 손에 밀가루가 묻어있으면 화면 터치를 못 한다. 매번 화면을 깨우려고 손을 씻을 수는 없다.

Wake Lock API로 화면 꺼짐을 방지한다.

## Wake Lock API

```typescript
let wakeLock = null;

async function requestWakeLock() {
  wakeLock = await navigator.wakeLock.request('screen');
}

function releaseWakeLock() {
  wakeLock?.release();
  wakeLock = null;
}
```

`navigator.wakeLock.request('screen')`을 호출하면 화면이 꺼지지 않는다. 요리가 끝나면 `release()`로 해제. 전등 스위치를 켜두는 거다. 요리가 끝나면 끈다.

## 앱 전환 시 재획득

핵심 주의점. 사용자가 다른 앱으로 전환했다가 돌아오면 Wake Lock이 자동으로 해제된다. 돌아올 때 다시 획득해야 한다.

```typescript
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible' && isCooking) {
    requestWakeLock();
  }
});
```

`visibilitychange` 이벤트로 감지. 페이지가 다시 보이면(`visible`) Wake Lock을 재요청. 집을 나갈 때 전등이 자동으로 꺼지는 센서등이라, 다시 들어오면 직접 켜야 한다.

## useEffect 정리

컴포넌트가 언마운트될 때 Wake Lock을 반드시 해제해야 한다. 안 그러면 요리 모드를 나갔는데도 화면이 안 꺼진다.

```typescript
useEffect(() => {
  requestWakeLock();

  const handleVisibility = () => {
    if (document.visibilityState === 'visible') {
      requestWakeLock();
    }
  };
  document.addEventListener('visibilitychange', handleVisibility);

  return () => {
    releaseWakeLock();
    document.removeEventListener('visibilitychange', handleVisibility);
  };
}, []);
```

cleanup 함수에서 `release()` + `removeEventListener()`. 퇴근할 때 에어컨 끄고 창문 닫는 거다.

## Graceful Degradation

Wake Lock API를 지원하지 않는 브라우저가 있다.

```typescript
if ('wakeLock' in navigator) {
  wakeLock = await navigator.wakeLock.request('screen');
}
```

optional chaining이나 `in` 연산자로 확인. 지원 안 하면 그냥 기본 동작(화면 자동 꺼짐)으로 동작한다. 에어컨이 없으면 선풍기라도 돌리는 게 아니라, 그냥 창문을 여는 거다. 핵심 기능이 아니니까 대체재를 만들 필요 없다.

## 브라우저 지원

| 브라우저 | 지원 |
|---------|------|
| Chrome 84+ | O |
| Safari 16.4+ | O |
| Firefox 126+ | O |
| 토스 앱 WebView | O (Chrome 기반) |

토스 앱이 Chrome 기반 WebView라 문제없다. 그래도 방어 코드는 넣어둔다.
