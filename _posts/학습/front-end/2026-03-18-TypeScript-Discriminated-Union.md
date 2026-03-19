---
title: "[학습] TypeScript Discriminated Union — 타입으로 분기 처리하기"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-18 20:10:00 +0900
categories: [학습, Frontend]
tags: [학습, TypeScript, 타입 시스템, Discriminated Union]
render_with_liquid: false
---

## 왜 필요했나

[어드민 FE](/posts/API-클라이언트-인프라-구축/)에서 BE 응답 타입을 정의하는데 문제가 있었다. 처음에 이렇게 만들었다:

```typescript
interface ApiResponse<T> {
  success: boolean;
  data: T | null;
  error: ApiErrorPayload | null;
}
```

이러면 `success`가 `true`인데 `data`가 `null`인 상태가 타입적으로 허용된다. `success`가 `false`인데 `error`가 `null`인 것도 마찬가지. 타입 시스템이 잘못된 상태를 허용하고 있다.

## Discriminated Union이 뭔지

**판별 유니온**이라고도 한다. 공통 필드(판별자)의 **리터럴 값**으로 타입을 분기하는 패턴이다.

```typescript
type ApiResponse<T> =
  | { success: true;  data: T;    error: null }
  | { success: false; data: null; error: ApiErrorPayload };
```

`success` 필드가 판별자다. `true`면 `data`는 반드시 `T`이고 `error`는 반드시 `null`. `false`면 반대. **불가능한 상태가 타입 레벨에서 존재할 수 없다.**

```typescript
function handle(res: ApiResponse<Recipe>) {
  if (res.success) {
    // TypeScript가 res.data를 Recipe로 추론
    console.log(res.data.title);
  } else {
    // TypeScript가 res.error를 ApiErrorPayload로 추론
    console.log(res.error.code);
  }
}
```

`if (res.success)` 한 줄로 타입이 좁혀진다(type narrowing). 캐스팅 없이.

## 리터럴 유니온 타입

Discriminated Union과 같이 쓰는 패턴이 하나 더 있다. [TasteNote](/posts/약관과-Firebase-수정/)에서 약관 타입을 정의할 때 적용했다.

```typescript
// before — 아무 문자열이나 들어감
interface Term {
  type: string;
}

// after — 허용된 값만 들어감
type TermType = 'TERMS_OF_SERVICE' | 'PRIVACY_POLICY';
interface Term {
  type: TermType;
}
```

`string`을 쓰면 `"SERVCE"`(오타)가 들어가도 컴파일 에러가 안 난다. 리터럴 유니온을 쓰면 **허용된 값 이외에는 컴파일 타임에 잡힌다.**

### 타입 가드 함수

URL 파라미터처럼 런타임에 들어오는 값은 컴파일 타임에 검증이 안 된다. 이때 타입 가드 함수를 쓴다.

```typescript
function isTermType(value: string): value is TermType {
  return value === 'TERMS_OF_SERVICE' || value === 'PRIVACY_POLICY';
}

// 사용
const type = useParams().termType; // string
if (isTermType(type)) {
  // 여기서 type은 TermType으로 좁혀짐
  fetchTermByType(type);
} else {
  // 404 UI
}
```

`value is TermType`이 핵심이다. 이게 TypeScript에게 "이 함수가 `true`를 반환하면 `value`는 `TermType`이야"라고 알려주는 거다. 일반 `boolean` 반환과 다르게 타입이 좁혀진다.

## Java의 sealed interface랑 비교

BE에서 비슷한 패턴이 있다. Java 17의 sealed interface.

```java
sealed interface ApiResult<T> {
  record Success<T>(T data) implements ApiResult<T> {}
  record Failure(ErrorCode code, String message) implements ApiResult<Void> {}
}
```

```typescript
type ApiResult<T> =
  | { type: 'success'; data: T }
  | { type: 'failure'; code: string; message: string };
```

둘 다 **"가능한 상태를 타입으로 제한한다"**는 목적이 같다. Java는 `instanceof`로, TypeScript는 판별자 필드로 분기한다.

| | TypeScript Discriminated Union | Java sealed interface |
|---|---|---|
| 분기 방식 | 판별자 필드 값 비교 | `instanceof` 패턴 매칭 |
| 완전성 검사 | `switch`에서 `never` 타입 | `switch`에서 컴파일 에러 |
| 추가 시 | 유니온에 타입 추가 | `permits`에 클래스 추가 |

## 어디에 쓰면 좋은지

- **API 응답**: 성공/실패 분기 (이번에 적용한 케이스)
- **상태 머신**: `'idle' | 'loading' | 'success' | 'error'` 같은 UI 상태
- **이벤트 처리**: `{ type: 'click', x: number } | { type: 'key', key: string }`
- **폼 필드 타입**: `{ kind: 'text', value: string } | { kind: 'number', value: number }`

공통점은 **"이 값이 A면 이런 필드가 있고, B면 저런 필드가 있다"**는 구조다.

## 정리

핵심은 한 가지다. **불가능한 상태를 타입 시스템으로 표현 불가능하게 만들어라.** `string`을 쓸 수 있는 곳에 리터럴 유니온을 쓰고, `T | null`을 쓸 수 있는 곳에 판별 유니온을 쓰면 런타임 에러가 컴파일 타임으로 올라온다.

"타입 시스템으로 잡을 수 있는 건 런타임까지 미루지 않는다" — 이번 어드민 작업에서 제일 많이 한 생각이다.
