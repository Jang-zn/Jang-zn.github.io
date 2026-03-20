---
title: "[학습] TanStack Query 캐시 키 — 키 설계가 곧 캐시 설계다"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-18 20:20:00 +0900
categories: [학습, Frontend]
tags: [학습, TanStack Query, React Query, 캐시, queryKey]
render_with_liquid: false
---

## 왜 필요했나

[TasteNote](/posts/약관과-Firebase-수정/)에서 약관 기능을 만들다가 버그가 나왔다. `useTermByType('TERMS_OF_SERVICE')`로 이용약관을 조회하고, `useTermDetail(termId)`로 같은 약관을 상세 조회했는데 **잘못된 데이터가 나왔다.** 원인은 queryKey 충돌.

## TanStack Query 캐시가 동작하는 방식

TanStack Query(구 React Query)는 서버 상태를 캐시에 저장하고, **queryKey가 같으면 같은 데이터**로 취급한다. 이게 전부다.

```typescript
// 이 두 쿼리는 같은 캐시를 공유한다
useQuery({ queryKey: ['terms', type], queryFn: fetchByType });
useQuery({ queryKey: ['terms', id],   queryFn: fetchById });
```

`type`이 `'TERMS_OF_SERVICE'`이고 `id`가 `'TERMS_OF_SERVICE'`... 가 아니더라도, 배열 구조가 비슷하면 시리얼라이즈 결과가 겹칠 수 있다. TanStack Query는 queryKey를 **JSON.stringify 기반으로 직렬화**해서 비교하기 때문에 `['terms', 'abc123']`과 `['terms', 'abc123']`은 같은 캐시다.

## 실제로 터진 문제

```typescript
// 충돌 발생
const useTermByType = (type: TermType) =>
  useQuery({ queryKey: ['terms', type], ... });

const useTermDetail = (termId: string) =>
  useQuery({ queryKey: ['terms', termId], ... });
```

두 훅 모두 queryKey가 `['terms', 문자열]` 형태다. `type`과 `termId` 값이 우연히 겹치지 않더라도, **의미가 다른 데이터가 같은 네임스페이스를 공유**하는 것 자체가 문제다. 아파트 호수를 "301호"라고만 적어두면, A동 301호인지 B동 301호인지 구분이 안 되는 것과 같다. 캐시 무효화할 때 `queryClient.invalidateQueries({ queryKey: ['terms'] })`를 하면 양쪽 다 날아간다.

## 해결: 의미 단위로 키 분리

```typescript
// 수정 후
const useTermByType = (type: TermType) =>
  useQuery({ queryKey: ['terms', 'byType', type], ... });

const useTermDetail = (termId: string) =>
  useQuery({ queryKey: ['terms', 'byId', termId], ... });
```

두 번째 요소에 **조회 방식**을 넣어서 네임스페이스를 분리했다. 이러면:
- `['terms', 'byType', 'TERMS_OF_SERVICE']` — type으로 조회
- `['terms', 'byId', 'abc-123']` — id로 조회

절대 겹치지 않는다.

## queryKey 설계 패턴

BE에서 Redis 캐시 키를 설계하는 것과 비슷하다. [다층 캐싱 포스트](/posts/다층-캐싱/)에서 `parse-result:{urlHash}`, `recipe:{recipeId}` 같은 키를 설계한 것처럼, 프론트에서도 캐시 키 네이밍 규칙이 필요하다.

### 1. 계층 구조로 설계

```typescript
// 패턴: [엔티티, 조회방식, 파라미터]
['users']                        // 유저 목록
['users', 'byId', userId]       // 유저 상세
['users', 'byId', userId, 'recipes'] // 유저의 레시피 목록

['recipes']                      // 레시피 목록
['recipes', 'byId', recipeId]   // 레시피 상세
['recipes', 'search', keyword]  // 레시피 검색
```

### 2. 무효화 범위 제어

계층 구조의 장점은 무효화 범위를 선택할 수 있다는 것.

```typescript
// 유저 관련 캐시 전부 날리기
queryClient.invalidateQueries({ queryKey: ['users'] });

// 특정 유저만 날리기
queryClient.invalidateQueries({ queryKey: ['users', 'byId', userId] });

// 검색 결과만 날리기
queryClient.invalidateQueries({ queryKey: ['recipes', 'search'] });
```

BE의 캐시 무효화와 같은 고민이다. 범위가 넓으면 불필요한 재요청이 늘고, 좁으면 stale 데이터가 남는다.

### 3. queryKey Factory 패턴

키를 문자열로 하드코딩하면 오타가 나고 관리가 안 된다. Factory로 뽑아낸다.

```typescript
const termKeys = {
  all:    () => ['terms'] as const,
  byType: (type: TermType) => ['terms', 'byType', type] as const,
  byId:   (id: string) => ['terms', 'byId', id] as const,
};

// 사용
useQuery({ queryKey: termKeys.byType('TERMS_OF_SERVICE'), ... });
queryClient.invalidateQueries({ queryKey: termKeys.all() });
```

타입 안전하고, 키 구조가 바뀌면 한 곳만 수정하면 된다.

## 조건부 쿼리 — enabled

약관 상세 페이지에서 URL 파라미터가 유효하지 않으면 쿼리를 아예 실행하지 않아야 한다. `enabled` 옵션을 쓴다.

```typescript
const useTermByType = (type: string) =>
  useQuery({
    queryKey: termKeys.byType(type as TermType),
    queryFn: () => fetchByType(type as TermType),
    enabled: isTermType(type), // false면 쿼리 실행 안 함
  });
```

`enabled: false`면 네트워크 요청 자체가 안 나간다. 유효하지 않은 파라미터로 API를 때리는 걸 방지한다. [TypeScript 타입 가드](/posts/TypeScript-Discriminated-Union/)와 같이 쓰면 런타임 검증과 타입 좁히기를 동시에 할 수 있다.

## BE 캐시 키와 비교

| | FE (TanStack Query) | BE (Redis) |
|---|---|---|
| 키 형식 | 배열 `['terms', 'byType', type]` | 문자열 `terms:byType:{type}` |
| 충돌 방지 | 배열 요소로 네임스페이스 분리 | 콜론(`:`)으로 네임스페이스 분리 |
| TTL | `staleTime`, `gcTime` | `@Cacheable(ttl=...)` |
| 무효화 | `invalidateQueries` (prefix 매칭) | `evict` (패턴 매칭) |
| 스탬피드 방지 | `staleTime` 동안 캐시 서빙 | [분산락 + Double-Check](/posts/캐시-스탬피드/) |

구조는 다르지만 **"키 설계가 곧 캐시 설계"**라는 건 같다. 키가 엉망이면 캐시가 엉망이 된다.

## 정리

1. **queryKey는 의미 단위로 분리**: `['entity', '조회방식', '파라미터']`
2. **충돌 확인**: 같은 엔티티를 다른 방식으로 조회하면 반드시 키를 분리
3. **Factory 패턴**: 키를 하드코딩하지 말고 함수로 뽑아내기
4. **무효화 범위**: 계층 구조로 설계해서 선택적으로 날릴 수 있게
5. **enabled**: 유효하지 않은 파라미터면 쿼리 자체를 막기

이번에 queryKey 충돌로 버그가 나고 나서야 "아 이것도 설계해야 하는 거구나" 깨달았다. BE에서 Redis 캐시 키를 신경 써서 설계하는 것처럼 FE 캐시 키도 마찬가지다.
