---
title: "[학습] location.state와 페이지간 데이터 전달"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-26 14:00:00 +0900
categories: [학습, Frontend]
tags: [학습, React, Router, location.state, 네비게이션]
difficulty: ⭐4
render_with_liquid: false
---

## 다음 페이지에 데이터를 넘기는 4가지 방법

TasteNote 요리 모드 완료 후 새 버전 페이지로 이동할 때, 요리 중 변경한 재료/단계 데이터를 넘겨야 했다. 페이지간 데이터 전달 방법을 비교한다.

## 방법 비교

| 방법 | 데이터 크기 | URL 노출 | 새로고침 유지 | 적합한 경우 |
|------|-----------|---------|-------------|-----------|
| URL params | 작음 | O | O | ID, 슬러그 |
| Query string | 작~중 | O | O | 필터, 검색어 |
| location.state | 중~대 | X | △ | 임시 전달 데이터 |
| 전역 store | 제한 없음 | X | O | 여러 페이지에서 공유 |

## location.state

React Router의 `navigate()`에 `state`를 넘길 수 있다.

```typescript
navigate('/new-version', {
  state: {
    fromCooking: true,
    cookingEdits: { modifiedIngredients, modifiedSteps, memo },
  }
});
```

받는 쪽에서:

```typescript
const location = useLocation();
const { fromCooking, cookingEdits } = location.state ?? {};
```

URL에 노출되지 않는다. 사용자가 주소를 복사해도 데이터가 포함되지 않는다. 편지를 봉투에 넣어서 보내는 거다. 겉봉(URL)에는 주소만 적고, 내용물(state)은 안에 들어있다.

### 한계

- 새로고침하면 state가 사라질 수 있다 (브라우저에 따라 다름)
- 뒤로 가기로 돌아왔을 때 state가 유지될 수도 있다
- 직접 URL 입력으로 접근하면 state가 없다

그래서 `location.state`는 "있으면 쓰고 없으면 기본값"으로 처리해야 한다. 없을 때도 페이지가 정상 동작해야 한다.

## NavigationAdapter 확장

TasteNote는 `NavigationAdapter` 인터페이스로 라우팅을 추상화한다. 토스 앱 내부 네비게이션과 웹 브라우저 네비게이션을 같은 인터페이스로.

```typescript
interface NavigationAdapter {
  push(screen: ScreenName, params?: Record<string, string>, options?: { state?: unknown }): void;
  replace(screen: ScreenName, params?: Record<string, string>, options?: { state?: unknown }): void;
}
```

`options` 파라미터를 추가해서 state를 전달할 수 있게 했다. 기존 코드는 `options`를 안 넘기면 되니까 하위 호환성이 유지된다.

## Fire-and-forget 패턴

요리 로그는 서버에 저장하지만 응답을 기다리지 않는다.

```typescript
// 서버에 보내고 결과를 기다리지 않음
api.postCookingLog(logData);
navigate('/recipe-detail');
```

로그는 "보내면 끝"이다. 실패해도 사용자 경험에 영향이 없다. 편지를 우체통에 넣고 돌아서는 거다. 배달 확인을 기다리며 우체통 앞에 서 있을 필요 없다.

## 프로젝트 적용

요리 모드 → 새 버전 페이지 전환에 `location.state` 적용:

```
요리 완료 → CookingCompleteSheet → "새 버전 저장"
→ navigate('/new-version', { state: { fromCooking: true, cookingEdits } })
→ NewVersionPage에서 state 읽어서 폼 프리필
```

`fromCooking`이 true면 요리 중 변경한 재료/단계가 폼에 미리 채워진다. false면(직접 접근) 빈 폼으로 시작.
