---
title: "[학습] Toast, Snackbar — 사용자 피드백 UX와 Feedback Gap"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-25 20:00:00 +0900
categories: [학습, 기획]
tags: [학습, React, Toast, Snackbar, UX, 피드백]
difficulty: ⭐2
render_with_liquid: false
---

## 버튼을 눌렀는데 아무 반응이 없으면

사용자는 다시 누른다. 아니면 "안 됐나?" 하고 뒤로 갔다 들어온다. 이게 Feedback Gap이다.

---

## Feedback Gap

**정의**: 사용자가 행동을 취한 후 시스템 반응이 오기까지의 공백. 이 공백이 인지되면 불안감이 생긴다.

0.1초 이내: 즉각 반응처럼 느낌
0.3초 초과: "됐나?" 의식이 생기기 시작
1초 초과: 명시적 로딩 표시가 없으면 사용자는 "안 됐다"고 판단하고 재시도

비동기 작업(API 호출, 파싱 등)은 구조상 Feedback Gap이 발생한다. 이를 메우는 도구가 Toast와 Snackbar다.

---

## Toast vs Snackbar

같은 알림처럼 보이지만 목적이 다르다.

| | Toast | Snackbar |
|---|---|---|
| 용도 | 즉각 결과 알림 | 진행 상태 + 결과 |
| 지속 시간 | 2~3초 자동 사라짐 | 작업 완료까지 유지 |
| 위치 | 상단 | 하단 |
| 상호작용 | 없음 (읽기만) | 탭하면 상세 이동 가능 |

식당 비유: "주문 접수됐습니다" → Toast. 주방 화면의 "3번 테이블 조리 중" → Snackbar.

**Toast를 써야 할 때**: 저장 완료, 삭제 완료, 전송 완료 등 단순 결과 확인.
**Snackbar를 써야 할 때**: 백그라운드 작업이 진행 중이고, 완료 후 다른 화면으로 이동 가능할 때.

---

## Toast 위치: 상단이어야 하는 이유

습관적으로 하단에 두는 경우가 많다. 모바일 실기기로 테스트하면 문제가 드러난다.

모바일 하단은 엄지 조작 영역이다. Toast가 손에 가려진다. 노치/Dynamic Island 기기는 상단도 고려해야 한다.

```css
top: max(env(safe-area-inset-top), 12px);
```

`env(safe-area-inset-top)`: 노치, Dynamic Island 등 시스템 UI가 차지하는 영역. 값이 있는 기기에서는 그 아래부터 표시. 없는 기기에서는 0이 되니까 `max`로 최소값 보장.

---

## 피드백 색상: 성공/실패 분리

같은 색상으로 성공과 에러를 표시하면 신호가 없는 신호등이다. 사용자는 텍스트를 읽기 전에 색으로 결과를 판단한다.

성공 → 녹색 계열
에러 → 붉은색 계열
정보 → 파란색 계열

개발 중에는 인지하기 어렵다. 내가 만들었으니까 텍스트를 읽는다. 에러 메시지가 녹색으로 뜨는 걸 QA 단계에 가서야 발견하는 이유가 이거다.

---

## Snackbar와 staleTime의 관계

Snackbar가 떠있는 동안 그 뒤 목록이 갱신되면 레이아웃이 바뀐다. 사용자가 Snackbar를 읽고 있는 상태에서 화면이 움직이는 거다.

React Query의 `staleTime`은 캐시를 얼마나 "신선"하게 유지할지 결정한다. Snackbar 표시 시간보다 `staleTime`이 짧으면 Snackbar가 살아있는 동안 목록이 리페치될 수 있다.

Snackbar 지속 시간 > staleTime이면 이 문제가 생긴다. Snackbar 타이머보다 충분히 긴 staleTime을 설정하거나, Snackbar 활성 상태일 때 리페치를 막는 방식으로 해결한다.

---

## TasteNote 적용

레시피 등록 결과 화면이 두 가지 경로에서 달랐다. 캐시 히트(즉시)와 SSE 파싱(5~15초)이 서로 다른 UI로 처리됐다.

같은 결과인데 다른 피드백은 일관성이 없다. 두 경로 모두 홈 Snackbar로 통일했다. 캐시 히트도 홈으로 이동 후 Snackbar 표시 → 상세 이동 플로우로 맞췄다.

레시피 수정 완료 시 Toast가 없어서 Feedback Gap이 있었다. "저장됐나?" 확인하러 뒤로 갔다 들어오는 상황. `onAllDone` 콜백에 `showToast` 한 줄로 해결.

상세 구현은 [토스트 통일과 인앱결제 UI](/posts/토스트-통일과-인앱결제-UI/) 참고.
