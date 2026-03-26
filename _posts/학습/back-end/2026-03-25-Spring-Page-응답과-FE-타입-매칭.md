---
title: "[학습] Spring Page 응답과 FE 타입 매칭"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-25 14:00:00 +0900
categories: [학습, Backend]
tags: [학습, Spring, Page, TypeScript, API, 페이지네이션]
difficulty: ⭐4
render_with_liquid: false
---

## 배열인 줄 알았더니 객체였다

[앱인토스 Admin에서 대시보드 진입 시 에러가 쏟아졌다](/posts/BE-FE-API-불일치-전면-수정/). `activities.map is not a function`. 배열이 아닌 걸 배열처럼 쓰니까 터진 거다. 원인은 Spring의 `Page` 응답 구조를 FE에서 몰랐기 때문.

## Spring의 Page 객체

Spring Data JPA에서 `findAll(Pageable)`을 호출하면 `Page<T>` 객체가 반환된다. 이걸 그대로 JSON으로 직렬화하면 이런 구조가 된다.

```json
{
  "content": [...],
  "pageable": { "pageNumber": 0, "pageSize": 20 },
  "totalElements": 150,
  "totalPages": 8,
  "number": 0,
  "size": 20,
  "first": true,
  "last": false,
  "empty": false
}
```

핵심은 `content`에 실제 데이터가 들어있다는 것. 최상위가 배열이 아니라 객체다. 식당에서 음식이 접시에 담겨 나오는데, FE는 트레이(객체) 없이 접시(배열)만 올 거라고 기대한 거다.

## FE에서 흔한 실수

### 실수 1: 응답을 배열로 기대

```typescript
// 이렇게 정의하면
type ApiResponse<T> = {
  data: T[];
  page: number;
  total: number;
}

// 서버 응답은 이건데
{ content: [...], number: 0, totalElements: 150 }

// data가 undefined → .map() 터짐
```

### 실수 2: 필드명 불일치

| FE 타입 | Spring 실제 | 비고 |
|---------|------------|------|
| `data` | `content` | 데이터 배열 |
| `page` | `number` | 현재 페이지 (0-based) |
| `total` | `totalElements` | 전체 건수 |
| `totalPages` | `totalPages` | 이건 같음 |
| `perPage` | `size` | 페이지당 건수 |

### 실수 3: 1-based vs 0-based

Spring Page는 0-based. 첫 페이지가 0이다. FE에서 "1페이지"를 요청하려면 `page=0`을 보내야 한다. `page=1`을 보내면 두 번째 페이지가 온다.

## 올바른 FE 타입 정의

Spring Page 구조에 맞춘 타입:

```typescript
interface PaginatedResponse<T> {
  content: T[];
  number: number;        // 현재 페이지 (0-based)
  size: number;          // 페이지당 건수
  totalElements: number; // 전체 건수
  totalPages: number;    // 전체 페이지 수
  first: boolean;
  last: boolean;
}
```

커스텀 래퍼를 만들어서 Spring Page 구조를 변환하는 방법도 있다. BE에서 `PageResponse<T>` DTO를 만들어서 필드명을 FE 친화적으로 바꾸는 거다. 하지만 이번 프로젝트에서는 FE를 Spring 구조에 맞추는 게 더 빨랐다. BE도 FE도 내가 개발하니까.

## Request Interceptor 토큰 문제

이번에 같이 발견된 문제. Axios request interceptor가 모든 요청에 토큰을 붙이는데, 로그인 요청에도 만료된 토큰이 붙었다.

```typescript
// Before: 모든 요청에 토큰
interceptor(config) {
  config.headers.Authorization = `Bearer ${token}`;
  return config;
}

// After: 공개 엔드포인트 제외
const PUBLIC_ENDPOINTS = ['/auth/login', '/auth/refresh'];
interceptor(config) {
  if (PUBLIC_ENDPOINTS.some(ep => config.url?.includes(ep))) {
    return config;
  }
  config.headers.Authorization = `Bearer ${token}`;
  return config;
}
```

토큰 저장소도 `localStorage` 직접 접근에서 `useAuthStore.getState()`로 변경. Zustand store가 single source of truth가 되도록.

## 방어적 데이터 접근

API 응답이 항상 정상이라는 가정은 위험하다. 네트워크 에러, 서버 에러, 빈 응답 — 비정상 케이스에서 FE가 안 터지려면 optional chaining이 필수.

```typescript
// Bad: 한 곳이라도 null이면 전체 터짐
data.content.map(item => ...)

// Good: 체이닝으로 방어
data?.content?.map(item => ...) ?? []
```

catch 블록에서 상태 리셋도 중요하다. 에러가 났는데 이전 데이터가 그대로 표시되면 사용자가 "정상"으로 오해한다.

## 정리

| 교훈 | 내용 |
|------|------|
| FE 타입은 실제 응답 기준 | "아마 이렇겠지"로 만들면 런타임에서 터진다 |
| Spring Page는 객체 | 배열이 아님. `content` 필드에 데이터 |
| 0-based 페이징 | 첫 페이지가 0. FE에서 변환 필요 |
| interceptor 예외 처리 | 공개 엔드포인트는 토큰 제외 |
| optional chaining | API 응답을 신뢰하지 말 것 |

BE와 FE를 동시에 개발하면서 "나중에 맞추면 되지"라고 미루면 이렇게 한꺼번에 터진다. Swagger나 실제 응답을 보면서 FE 타입을 정의하는 게 결국 더 빠르다.
