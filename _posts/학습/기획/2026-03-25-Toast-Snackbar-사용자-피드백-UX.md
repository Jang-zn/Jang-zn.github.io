---
title: "[학습] 사용자 피드백 UX — Feedback Gap, Toast, Snackbar, Optimistic UI"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-25 20:00:00 +0900
categories: [학습, 기획]
tags: [학습, React, Toast, Snackbar, UX, 피드백, OptimisticUI]
difficulty: ⭐2
render_with_liquid: false
---

## 사용자가 "됐나, 안 됐나" 모르면 다시 누른다

피드백이 없는 인터페이스는 벽에 대고 말하는 것과 같다. 아무 반응 없으면 사용자는 행동을 반복하거나, 이탈하거나, 뭔가 잘못됐다고 판단한다.

---

## 시스템 상태의 가시성(Visibility of System Status) — Nielsen의 첫 번째 원칙

**정의**: 시스템이 지금 무슨 상태인지 사용자에게 항상 알려줘야 한다.

**출처**: Nielsen, J. (1994). *"10 Usability Heuristics for User Interface Design"* — Nielsen Norman Group. Heuristic #1.

"항상 합리적인 시간 안에 적절한 피드백을 통해 사용자에게 현재 상태를 알려라."

10가지 원칙 중 1번. 그게 우선순위를 말해준다. 상태 가시성이 없으면 나머지 원칙도 의미가 없다.

적용 범위:
- 버튼 클릭 → 즉각 시각 반응 (눌림 효과, 로딩 스피너)
- 비동기 작업 → 진행 중임을 알리는 상태 표시
- 완료 → 성공/실패 명확히 구분

---

## Feedback Gap과 Response Time 연구

**Feedback Gap 정의**: 사용자가 행동을 취한 후 시스템 반응이 오기까지의 공백.

**Response Time 기준**: Nielsen, J. (1993). *"Response Times: The 3 Important Limits"*. 원래 Miller(1968)가 제안한 0.1/1/10초 프레임워크를 UI에 적용.

| 한계 | 기준 | 사용자 경험 |
|------|------|-----------|
| 0.1초 | 즉각 반응 | "내가 한 거야" 느낌. 피드백 불필요. |
| 1초 | 흐름 유지 | 생각의 흐름이 끊기지 않는 한계. 스피너 불필요하지만 있으면 좋음. |
| 10초 | 주의 유지 한계 | 이 이상이면 사용자는 딴 짓을 시작하거나 이탈. 진행 표시 필수. |

모바일 API 호출의 평균 RTT는 300ms~2s다. "1초 이내"에 들어오는 경우도 있지만 네트워크 상태에 따라 달라진다. 항상 피드백을 준다고 보수적으로 설계하는 게 낫다.

**Feedback Gap이 문제가 되는 순간**: 버튼을 눌렀는데 0.3초 이상 아무 변화가 없으면 사용자는 "안 됐나?" 생각하고 다시 누른다. 중복 요청이 발생하는 이유가 여기에 있다.

---

## Toast vs Snackbar

같은 "알림"처럼 보이지만 목적이 다르다.

| | Toast | Snackbar |
|---|---|---|
| 목적 | 즉각 결과 알림 | 진행 상태 + 결과 |
| 지속 시간 | 2~4초 자동 소멸 | 작업 완료까지 유지 가능 |
| 위치 | 상단 (모바일 기준) | 하단 |
| 상호작용 | 없음 | 탭 → 상세 이동 |
| 사용 케이스 | 저장 완료, 삭제, 전송 | 백그라운드 작업 진행 중 |

비유: 식당 직원이 "주문 접수됐습니다"라고 말하는 게 Toast. 주방 위 화면에 "3번 테이블 조리 중"이 계속 떠있는 게 Snackbar.

**Android Material Design**은 둘을 명확히 구분한다. Toast는 시스템 레벨, Snackbar는 앱 레벨. iOS는 Snackbar 개념이 없고 Banner(알림 배너)가 유사 역할을 한다.

---

## Toast 위치: 왜 상단인가

"하단이 더 보기 편하다"는 생각이 있다. 엄지가 거기 있으니까.

모바일에서 하단은 엄지 조작 영역이다. Toast가 손에 가려진다. 반응이 없는 게 아니라 보이지 않는 거다. 실기기 테스트 없이 에뮬레이터나 스크린샷으로만 확인하면 발견이 늦다.

상단으로 올리면 노치/Dynamic Island 문제가 생긴다. `safe-area-inset-top`으로 해결한다.

```css
top: max(env(safe-area-inset-top), 12px);
```

`env(safe-area-inset-top)`: iOS에서 노치 또는 Dynamic Island 높이를 반환하는 CSS 환경 변수. 해당 기기가 없으면 0. `max`로 감싸서 최소 12px 보장.

---

## 피드백 색상: 색만으로 의미를 전달하면 안 된다

성공/에러를 색상으로만 구분하는 설계는 두 가지 문제가 있다.

1. **색맹 접근성**: 전체 인구의 약 8%(남성 기준)가 적색-녹색 구분에 어려움을 겪는다. 색만으로 성공/에러를 구분하면 접근성 이슈다.
2. **정보 누락**: 색을 인식해도 맥락이 없으면 의미가 모호하다.

WCAG(Web Content Accessibility Guidelines) 1.4.1: "색상을 유일한 시각적 수단으로 정보를 전달해서는 안 된다."

실용적 해결: 색상 + 아이콘 + 텍스트를 세트로. 성공은 체크 아이콘 + 녹색 + "완료", 에러는 X 아이콘 + 붉은색 + "실패 이유".

---

## 낙관적 UI(Optimistic UI)

**정의**: 서버 응답을 기다리지 않고 UI를 먼저 성공 상태로 업데이트하는 패턴. 요청이 실패하면 롤백.

**예시**: 트위터 좋아요 버튼. 누르면 즉시 하트가 빨개지고, 서버와는 백그라운드에서 동기화. 실패하면 다시 원래 상태로.

**장점**: 0.1초 이내 반응 → "즉각 반응" 경험. Feedback Gap 제거.
**단점**: 실패 시 롤백 UX를 설계해야 한다. 실패가 드물고, 실패 시 사용자가 이해할 수 있는 피드백이 가능한 경우에 적합.

낙관적 UI가 적합한 경우:
- 성공률이 99%에 가까운 작업
- 실패해도 사용자가 다시 시도하기 쉬운 경우
- 지연이 사용자 경험에 크게 영향을 미치는 경우

적합하지 않은 경우:
- 결제, 계좌이체 같이 실패 시 심각한 영향
- 실패가 자주 발생하는 환경

---

## Skeleton UI vs Spinner

둘 다 "로딩 중"을 표시하지만 체감이 다르다.

| | Skeleton UI | Spinner |
|---|---|---|
| 원리 | 콘텐츠 구조를 미리 보여줌 | 진행 중임을 표시 |
| 체감 대기시간 | 짧아 보임 | 길어 보임 |
| 적합한 상황 | 콘텐츠 구조가 예측 가능할 때 | 완료 시간 불확실 |

Facebook이 Skeleton UI를 도입한 연구(2014): 같은 로딩 시간에서 Skeleton이 Spinner보다 체감 대기시간을 의미있게 줄였다.

**왜 짧아 보이나**: 콘텐츠가 들어올 공간이 이미 있으니까 레이아웃이 바뀌지 않는다. 레이아웃 변화(Content Layout Shift)가 없으면 완료까지 자연스럽게 이어진다.

**Spinner가 맞는 경우**: 완료 시간이 정말 불확실하거나, 화면 전체를 바꿔야 하는 작업(페이지 이동). 파일 업로드처럼 진행률을 퍼센트로 보여줄 수 있으면 Progress Bar가 더 좋다.

---

## 일시적 알림 vs 지속적 알림(Transient vs Persistent)

| | 일시적(Transient) | 지속적(Persistent) |
|---|---|---|
| 예시 | Toast, Snackbar | 배지, 상태 표시줄, 배너 |
| 지속 | 자동 소멸 | 사용자가 처리할 때까지 유지 |
| 적합 | 참고용 정보, 성공 알림 | 반드시 처리해야 하는 상태 |

읽지 않은 메시지는 지속적 알림(배지). "저장됐습니다"는 일시적 알림(Toast). 잘못된 사례: "에러가 발생했습니다"를 Toast로 2초 보여주는 것. 사용자가 못 읽고 사라지면 에러를 인지 못한다. 에러는 Persistent가 맞다.

---

## TasteNote 적용

- 캐시 히트(즉시)와 SSE 파싱(5~15초) 경로 모두 홈 Snackbar로 통일. 같은 결과에 다른 UI는 일관성 위반.
- 레시피 수정 완료 시 Feedback Gap 있었음 → `showToast` 한 줄로 해결.
- Toast 위치 하단 → 상단으로. 실기기에서 손에 가려지는 문제.
- 에러 Toast는 붉은색, 성공 Toast는 녹색으로 분리.
- Snackbar `staleTime`: 4초 타이머 동안 목록이 리페치되지 않도록 30초로 설정.

상세 구현은 [토스트 통일과 인앱결제 UI](/posts/토스트-통일과-인앱결제-UI/) 참고.

---

## 참고자료

- Nielsen, J. (1994). *10 Usability Heuristics*. Nielsen Norman Group
- Nielsen, J. (1993). *Response Times: The 3 Important Limits*. Nielsen Norman Group
- Miller, R.B. (1968). *Response Time in Man-Computer Conversational Transactions*. AFIPS Conference Proceedings
- WCAG 2.1, Criterion 1.4.1: Use of Color
- Material Design 3: *Snackbars*, *Dialogs* — m3.material.io
- Apple HIG: *Alerts, Banners* — developer.apple.com/design/human-interface-guidelines
