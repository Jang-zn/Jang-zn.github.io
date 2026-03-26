---
title: "[학습] BaseEntity 상속과 Command DTO — Spring 설계 판단 기준"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-25 16:00:00 +0900
categories: [학습, Backend]
tags: [학습, Spring, JPA, BaseEntity, Command, DTO, 설계]
render_with_liquid: false
---

## "다 상속하면 되는 거 아님?"

Spring 프로젝트에서 자주 보는 패턴 두 개. BaseEntity 상속과 Command DTO. 둘 다 "언제 쓰고 언제 안 쓰는지"를 판단하는 기준이 있다. [TasteNote BE](/posts/인앱결제-광고-제거-시스템/) 개발하면서 실제로 판단했던 기준을 정리한다.

## BaseEntity 상속

대부분의 Spring 프로젝트에 `BaseEntity`가 있다. `createdAt`과 `updatedAt`을 공통으로 들고 있는 추상 클래스.

### 판단 기준: "이 엔티티는 생성 후 수정되는가?"

| 엔티티 | 생성 후 수정? | BaseEntity 상속? |
|--------|-------------|-----------------|
| Recipe | O (제목, 재료 변경) | O |
| Purchase | O (상태 변경: ACTIVE→REFUNDED) | O |
| CookingLog | X (한번 기록하면 끝) | X |

CookingLog는 요리 기록이다. 한번 저장하면 수정할 일이 없다. `updatedAt`이 있으면 "이 기록이 수정된 적 있다"는 잘못된 신호를 줄 수 있다. 일기장에 "수정일" 칸이 있으면 "이거 고쳐 쓴 건가?" 의심하게 된다.

CookingLog는 `createdAt`만 직접 필드로 가진다. 불필요한 `updatedAt`을 제거.

### @MappedSuperclass

BaseEntity에 `@MappedSuperclass`를 붙이면 자식 엔티티의 테이블에 부모 필드가 그대로 들어간다. 별도 테이블이 생기지 않는다. `@Inheritance`와 다르다.

## Command DTO

GoF의 Command Pattern과 Spring에서 말하는 Command DTO는 다른 개념이다.

| | GoF Command Pattern | Command DTO |
|---|---|---|
| 목적 | 행위를 객체로 캡슐화 (execute/undo) | 파라미터를 하나로 묶기 |
| 예시 | `UndoableCommand.execute()` | `PurchaseGrantCommand(orderId, productType, amount)` |
| 사용처 | 작업 큐, 매크로 | 서비스 메서드 파라미터 |

### 왜 Command DTO를 쓰는가

서비스 메서드 파라미터가 4개 이상이면 가독성이 떨어진다.

```java
// Before: 파라미터 4개 — 뭐가 뭔지 호출부에서 안 보임
void grantProduct(Long userId, String orderId, String productType, int amount)

// After: Command DTO로 묶기
void grantProduct(PurchaseGrantCommand command)
```

택배 보낼 때 상자 하나에 물건을 담아서 보내는 거다. 물건 4개를 각각 손에 들고 가면 하나 떨어뜨린다.

### Request DTO와 Command DTO를 분리하는 이유

Controller에서 받는 Request DTO와 Service에 넘기는 Command DTO를 분리한다.

- Request DTO: API 스펙에 의존 (필드명, validation 어노테이션)
- Command DTO: 비즈니스 로직에 의존 (userId 추가, 내부 타입 변환)

Request를 그대로 Service에 넘기면 API 스펙이 바뀔 때 서비스 코드도 바뀐다. 현관문 잠금 장치를 바꿨는데 거실 리모컨까지 바꿔야 하면 구조가 잘못된 거다.

## 프로젝트 적용

Purchase 기능에서 둘 다 적용:

- `Purchase` 엔티티: BaseEntity 상속 (상태 변경이 있으므로)
- `PurchaseGrantCommand`: 구매 부여 시 userId, orderId, productType, amount를 묶음
- `RefundRequestCommand`: 환불 요청 시 userId, purchaseId, reason을 묶음

반면 CookingLog(0326에 추가 예정)는 BaseEntity를 상속하지 않고 `cookedAt` 필드만 가진다.
