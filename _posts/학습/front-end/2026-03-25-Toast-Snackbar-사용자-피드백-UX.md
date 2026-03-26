---
title: "[학습] Toast, Snackbar, SSE — 사용자 피드백 UX 통합"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-25 20:00:00 +0900
categories: [학습, Frontend]
tags: [학습, React, Toast, Snackbar, SSE, UX, 피드백]
render_with_liquid: false
---

## 사용자가 "됐나? 안 됐나?" 모르면 안 된다

[TasteNote FE](/posts/토스트-통일과-인앱결제-UI/)에서 알림 UX를 통합했다. Toast와 Snackbar가 뒤섞여 있었는데, 각각의 역할을 정리하고 통일했다. "피드백 공백"이 UX에서 가장 치명적인 실수다. 버튼을 눌렀는데 아무 반응이 없으면 사용자는 다시 누른다.

## Toast vs Snackbar

| | Toast | Snackbar |
|---|---|---|
| 용도 | 짧은 결과 알림 | 진행 상태 표시 |
| 지속 시간 | 2~3초 | 작업 완료까지 |
| 위치 | 상단 | 하단 |
| 사용자 행동 | 확인만 | 탭하면 상세 이동 |

Toast는 "저장 완료", "삭제 완료" 같은 즉각 결과. Snackbar는 "파싱 중..." 같은 비동기 진행 상태. 식당에서 "주문 접수됐습니다"가 Toast고, 주방 위 화면에 "3번 테이블 조리 중"이 Snackbar다.

## SSE와 캐시 히트 경로 통합

레시피 URL을 등록하면 두 가지 경로가 있다.

| 경로 | 소요 시간 | 기존 UI |
|------|----------|---------|
| 새 파싱 (SSE) | 5~15초 | 홈 하단 Snackbar |
| 캐시 히트 | 즉시 | 등록 페이지 오버레이 |

같은 "레시피 등록 완료"인데 화면이 달랐다. 캐시 히트도 홈 Snackbar로 통일했다.

`parseJobStore`에 `cachedRecipeId` 상태를 추가. 캐시 히트면 이 값을 세팅하고 홈으로 이동 → Snackbar가 4초 타이머 → 상세 이동. SSE 경로와 동일한 흐름.

### staleTime으로 타이밍 제어

Snackbar가 떠있는 동안 홈의 레시피 목록이 갱신되면 안 된다. 아직 상세로 이동하기 전인데 목록이 바뀌면 사용자가 혼란스럽다. TanStack Query의 `staleTime: 30000`(30초)으로 목록 갱신을 지연.

### 로딩 메시지 순환

SSE 파싱 중 "AI가 분석 중...", "재료를 읽는 중...", "단계를 정리하는 중..." 메시지가 순환한다. 간격은 4초. 2초면 너무 빠르고 산만하다. 실제 진행률은 모르지만, 메시지가 바뀌면 "뭔가 되고 있구나" 느낌을 준다. 엘리베이터 옆 거울과 같은 원리. 실제로 빨라지는 건 아닌데 기다리는 체감 시간이 줄어든다.

## Toast 색상 분리

기존에는 Toast가 전부 같은 색이었다. 성공이든 에러든. 신호등이 다 회색이면 정보가 없다.

```typescript
showToast('저장 완료', { type: 'success' }); // 녹색
showToast('저장 실패', { type: 'error' });   // 붉은색
```

## Toast 위치: 상단

모바일에서 하단은 엄지 영역이다. Toast가 하단에 뜨면 손에 가려진다. 상단으로 옮기고 `safe-area-inset-top`으로 노치/Dynamic Island 회피.

```css
top: max(env(safe-area-inset-top), 12px);
```

`safe-area-inset-top`은 노치가 있는 기기에서만 값이 있다. 없는 기기에서는 0이 되니까 `max`로 최소 12px 보장.

## 피드백 공백(Feedback Gap)

버튼을 눌렀는데 반응이 없는 시간. 이게 0.3초만 넘어도 사용자는 "안 됐나?" 생각한다. 레시피 수정 완료 시 Toast가 없어서 "저장됐나?" 확인하려고 뒤로 갔다 다시 들어오는 사용자가 있었다. `onAllDone`에 `showToast` 한 줄 추가로 해결.
