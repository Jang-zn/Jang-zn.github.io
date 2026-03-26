---
title: "[학습] 인앱결제 Bridge Adapter 패턴"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-25 19:00:00 +0900
categories: [학습, Frontend]
tags: [학습, React, Bridge, Adapter, 인앱결제, Toss, 디자인패턴]
difficulty: ⭐7
render_with_liquid: false
---

## Toss SDK는 토스 앱 안에서만 된다

[TasteNote 인앱결제 UI](/posts/토스트-통일과-인앱결제-UI/)를 만들면서 Bridge Adapter 패턴을 적용했다. Toss 미니앱에서 네이티브 기능(결제, 카메라 등)을 쓰려면 Bridge를 통해 SDK를 호출해야 한다. 문제는 이 SDK가 토스 앱 런타임에서만 존재한다는 것.

로컬 개발 환경에서는 `@apps-in-toss/web-bridge` 모듈이 없다. import하면 빌드가 터진다. 회사 사내 전화기가 회사에서만 쓸 수 있는 것처럼.

## Adapter 패턴으로 분리

`BridgeAdapter` 인터페이스를 정의하고, 환경별로 다른 구현체를 주입한다.

```typescript
interface BridgeAdapter {
  processProductGrant(sku: string): Promise<OrderResult>;
  completeProductGrant(orderId: string): Promise<void>;
  getPendingOrders(): Promise<PendingOrder[]>;
}
```

| 환경 | 구현체 | 동작 |
|------|--------|------|
| 토스 앱 | `TossBridgeAdapter` | 실제 SDK 호출 |
| 웹 브라우저 | `WebBridgeAdapter` | MSW 목 응답 |

## Dynamic Import

`TossBridgeAdapter`에서 Toss SDK를 dynamic import로 로드한다. 빌드 타임이 아니라 런타임에 모듈을 가져온다.

```typescript
async processProductGrant(sku: string) {
  const { purchase } = await import('@apps-in-toss/web-bridge');
  return purchase.processProductGrant({ sku });
}
```

빌드 시에는 이 import가 실행되지 않으니까 `@apps-in-toss/web-bridge`가 없어도 빌드가 성공한다. 냉장고 문을 열 때만 전등이 켜지는 거다. 문을 안 열면(토스 앱이 아니면) 전등이 없어도 상관없다.

## 구매 오케스트레이션

`usePurchaseFlow` 훅이 3단계 플로우를 관리한다.

```
1. Bridge → processProductGrant(sku)
   → Toss SDK가 결제 UI 표시, 사용자가 결제
2. Server → POST /api/v1/purchases/grant
   → 서버가 Toss API로 주문 검증 후 구매 부여
3. Bridge → completeProductGrant(orderId)
   → Toss SDK에 "서버 처리 완료" 알림
```

왜 3단계인가. 2단계에서 서버 응답이 안 오면 Toss SDK는 "미지급" 상태로 보관한다. 다음에 앱을 열면 `getPendingOrders()`로 미지급 건을 복원해서 2단계부터 재시도한다.

택배 수취 확인과 같다. 택배(결제)가 왔고 → 집주인(서버)이 확인했고 → 택배 기사(SDK)에게 "받았어요" 확인을 해줘야 다음 택배를 보낸다.

## WebBridgeAdapter (개발용)

로컬 개발 시에는 결제 SDK가 없으니까 목 구현체를 쓴다.

```typescript
class WebBridgeAdapter implements BridgeAdapter {
  async processProductGrant(sku: string) {
    // 고정 응답 반환
    return { orderId: 'mock-order-123', status: 'DONE' };
  }
}
```

MSW와 조합하면 서버 없이도 전체 구매 플로우를 테스트할 수 있다.

## 프로젝트 적용

인앱결제뿐 아니라 다른 Bridge 메서드(카메라, 공유 등)도 같은 패턴이다. `BridgeAdapter`에 메서드를 추가하고, 두 구현체에서 각각 구현한다. 인터페이스가 환경 차이를 숨겨준다.

[BE의 Port & Adapter](/posts/인앱결제-광고-제거-시스템/)와 같은 원리다. 도메인(비즈니스 로직)이 인프라(SDK, API)를 직접 모르게 하는 것.
