---
title: "[학습] Axios Interceptor — API 요청을 가로채서 전처리하기"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-18 20:00:00 +0900
categories: [학습, Frontend]
tags: [학습, Axios, Interceptor, API, TypeScript]
render_with_liquid: false
---

## 왜 필요했나

[어드민 FE](/posts/API-클라이언트-인프라-구축/)에서 BE API와 연동하는데, 매 요청마다 반복되는 작업이 있었다.

- 요청 보낼 때: camelCase → snake_case 변환, 인증 토큰 붙이기
- 응답 받을 때: snake_case → camelCase 변환, `ApiResponse<T>` 래퍼 벗기기, 에러 처리

이걸 API 호출하는 곳마다 넣으면 코드가 지저분해지고 실수가 나온다. Axios Interceptor로 한 곳에서 처리하면 된다.

## Interceptor가 뭔지

Axios에서 HTTP 요청이 나가기 전, 응답이 돌아온 후에 **가로채서 뭔가를 할 수 있는 훅**이다. Express의 미들웨어, Spring의 Filter/Interceptor랑 비슷한 개념.

```
[컴포넌트] → [Request Interceptor] → [서버]
[컴포넌트] ← [Response Interceptor] ← [서버]
```

Request Interceptor는 요청이 서버로 나가기 전에 실행되고, Response Interceptor는 서버 응답이 컴포넌트에 도달하기 전에 실행된다.

## 적용한 것들

### 1. snake_case ↔ camelCase 자동 변환

BE(Spring)에서 JSON 응답을 snake_case로 내려준다. Jackson에서 `SNAKE_CASE` 전략을 명시 설정한 것. FE는 TypeScript니까 camelCase가 표준이다. 이걸 매번 수동으로 바꾸면 실수가 나온다.

```typescript
// Request: 나갈 때 snake_case로
axios.interceptors.request.use(config => {
  if (config.data) {
    config.data = toSnakeCase(config.data);
  }
  return config;
});

// Response: 올 때 camelCase로
axios.interceptors.response.use(response => {
  response.data = toCamelCase(response.data);
  return response;
});
```

`toSnakeCase`, `toCamelCase`는 재귀 함수다. 중첩 객체, 배열 안의 객체까지 전부 변환한다. 한번 깔아놓으면 API 호출 코드에서 케이스를 신경 쓸 일이 없다.

주의할 점이 하나 있다. **토큰 리프레시 요청도 이 인터셉터를 탄다.** 리프레시 요청의 body(refresh_token)와 응답(access_token, refresh_token) 모두 변환이 적용되어야 한다. 인터셉터 안에서 별도 Axios 인스턴스를 만들어 리프레시 요청을 보내면 변환이 안 돼서 BE가 파싱을 못한다. 같은 인스턴스를 써야 한다.

### 2. ApiResponse 래퍼 언래핑

BE의 모든 응답이 이런 식으로 온다:

```json
{
  "success": true,
  "data": { "id": 1, "name": "김치찌개" },
  "error": null
}
```

매번 `response.data.data`로 꺼내는 건 번거롭다. Response Interceptor에서 자동으로 벗겨준다.

```typescript
axios.interceptors.response.use(response => {
  const body = response.data;
  if (body && typeof body === 'object' && 'success' in body) {
    response.data = body.data; // 래퍼를 벗기고 실제 데이터만
  }
  return response;
});
```

이러면 컴포넌트에서는 `response.data`만 쓰면 된다. 래퍼 구조를 몰라도 된다.

### 3. 에러 인터셉터

에러 처리가 가장 복잡했다. 세 가지 케이스를 한 곳에서 잡아야 한다.

**케이스 1: `success: false`인 200 응답**

HTTP 상태 코드는 200인데 비즈니스 로직에서 실패한 경우. BE가 `{ success: false, error: { code: "DUPLICATE", message: "이미 존재" } }` 같은 걸 보낸다. 이걸 안 잡으면 성공으로 처리된다.

```typescript
if (body.success === false) {
  throw new ApiError(
    body.error?.code ?? 'UNKNOWN_ERROR',
    body.error?.message ?? '알 수 없는 에러'
  );
}
```

`error`가 null일 수도 있다. `success: false`인데 `error` 없이 오는 케이스. 이것도 `UNKNOWN_ERROR`로 throw해야 한다. 안 그러면 실패 응답이 조용히 성공으로 넘어간다.

**케이스 2: 래퍼가 아닌 에러 응답**

BE가 항상 `ApiResponse` 래퍼로 주는 게 아니다. Spring Security에서 인증 실패하면 `{ message: "Unauthorized" }` 같은 구조화된 에러가 직접 온다. 이것도 `ApiError`로 변환해야 UI에서 일관되게 처리할 수 있다.

**케이스 3: 네트워크 에러**

서버 자체에 연결이 안 되는 경우. `response`가 없다. 이건 그냥 throw.

### 4. 토큰 리프레시

401이 오면 refresh token으로 새 access token을 받아서 원래 요청을 재시도한다. 이것도 Response Interceptor에서 처리.

여기서 중요한 건 **동시 요청 처리**다. 요청 A, B가 동시에 401을 받으면 리프레시가 2번 발생한다. 첫 번째 리프레시가 진행 중이면 나머지 요청은 Promise 큐에 넣어두고, 리프레시 완료 후 새 토큰으로 일괄 재시도한다.

## Interceptor 실행 순서

Interceptor가 여러 개 붙으면 순서가 중요하다.

```
Request:  등록 역순 (마지막에 등록한 게 먼저 실행)
Response: 등록 순서 (먼저 등록한 게 먼저 실행)
```

그래서 케이스 변환 → 래퍼 언래핑 → 에러 처리 순서로 등록해야 한다. 케이스 변환이 먼저 돼야 래퍼 필드명(`success`, `data`)이 camelCase로 바뀌어서 읽을 수 있다.

## Spring의 Filter/Interceptor랑 비교

BE를 하다 보면 Spring의 비슷한 개념들이 있다.

| 개념 | Axios Interceptor | Spring Filter | Spring Interceptor |
|------|-------------------|---------------|-------------------|
| 위치 | HTTP 클라이언트 | 서블릿 컨테이너 | Spring MVC |
| 실행 시점 | 요청 전/응답 후 | 요청 전/응답 후 | 컨트롤러 전/후 |
| 주 용도 | 토큰, 변환, 에러 | 인증, CORS, 로깅 | 권한, 로깅 |
| 등록 | `axios.interceptors.use()` | `@Component` | `WebMvcConfigurer` |

둘 다 **횡단 관심사를 한 곳에서 처리**한다는 점은 같다.

## 정리

한번 깔아놓으면 API 호출하는 쪽에서 신경 쓸 게 없어진다:

- 케이스 변환: 자동
- 래퍼 언래핑: 자동
- 에러 처리: 자동
- 토큰 갱신: 자동

이 인프라가 없으면 API 호출할 때마다 변환 코드, 에러 처리 코드가 반복된다. "인프라 코드는 한번 깔면 바꿀 일이 거의 없고, 비즈니스 코드는 매일 바뀐다" — 이 구분이 BE의 [클린 아키텍처](/posts/클린-아키텍처-인프라-추상화/)와 같은 맥락이다.
