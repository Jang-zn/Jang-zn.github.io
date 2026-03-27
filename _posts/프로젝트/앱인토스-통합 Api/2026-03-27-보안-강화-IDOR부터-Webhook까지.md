---
title: "[앱인토스-통합 Api] 보안 강화 — IDOR부터 Webhook 서명까지"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-27 18:00:00 +0900
categories: [프로젝트, 앱인토스-통합 Api]
tags: [프로젝트, 앱인토스-통합 Api, 보안, IDOR, JWT, RefreshToken, CORS, HMAC, Webhook]
render_with_liquid: false
---

## 기능은 다 만들었는데

[어제 V2 전체 스택](/posts/V2-냉장고-요리기록-전체-스택/)을 하루 만에 끝냈다. 냉장고, 요리기록, 인앱결제, 어드민 대시보드. 기능은 다 돌아간다. 문제는 보안이었다.

집을 다 지어놓고 보니 현관문에 잠금장치가 없는 상태. 창문도 열려있고, 뒷문은 아예 없었다.

Claude Code에 "보안 감사 해줘"라고 던졌다. 3개 분석 에이전트가 병렬로 돌아갔다. JWT 인증, API 컨트롤러 인가, 인프라 설정 — 각각 분야별로. 결과가 나왔다.

| 심각도 | 개수 | 주요 항목 |
|--------|------|----------|
| HIGH | 6개 | IDOR, Callback 무인증, JWT Secret 하드코딩, RT Rotation 미구현, Logout 부재, Webhook 서명 없음 |
| MEDIUM | 5개 | CORS 와일드카드, 보안 헤더 미설정, JWT Secret 미분리, Rate Limiting 부재, Secret 길이 검증 없음 |

HIGH 6개. 배포된 서버에서 이 상태로 돌고 있었다는 뜻이다.

## IDOR — 아파트 호수만 바꾸면 남의 집

IDOR(Insecure Direct Object Reference). OWASP Top 10에서 1위를 차지하는 Broken Access Control의 대표 취약점이다.

비유하면 이렇다. 아파트 로비에 경비원(로그인)은 있는데, 각 호수 문에 잠금장치가 없다. 301호 주민이 502호 문을 열고 들어가도 아무도 안 막는다.

우리 API가 정확히 이 상태였다. `/api/v1/recipes/abc`에서 `abc`를 다른 사용자의 레시피 ID로 바꾸면 수정도, 삭제도 됐다. 인증은 통과했지만 인가(소유권 검증)가 없었던 거다.

취약한 엔드포인트가 7개였다.

| Controller | 메서드 | 문제 |
|-----------|--------|------|
| RecipeController | updateRecipe | 타인 레시피 수정 가능 |
| RecipeController | deleteRecipe | 타인 레시피 삭제 가능 |
| PantryController | updateItem | 타인 냉장고 수정 가능 |
| PantryController | deleteItem | 타인 냉장고 삭제 가능 |
| RecipeVersionController | createVersion | 타인 레시피에 버전 추가 가능 |
| RecipeVersionController | updateVersion | 타인 레시피 버전 수정 가능 |
| CookingLogController | getLog | 타인 요리기록 조회 가능 |

수정 패턴은 단순하다. Controller에서 `@AuthenticationPrincipal`로 JWT에서 userId를 추출하고, Service에서 엔티티 소유자와 비교한다.

```java
// Controller — JWT에서 userId 추출
@AuthenticationPrincipal UserPrincipal principal
service.updateRecipe(id, principal.getUserId(), request);

// Service — 소유권 검증
if (!recipe.getUserId().equals(userId)) {
    throw new BusinessException(ErrorCode.FORBIDDEN);
}
```

7개 엔드포인트 전부 이 패턴을 적용했다. 한 곳이라도 빠지면 구멍이다. 소유권 검증에 대한 이론은 [학습 포스트](/posts/소유권-검증-엔티티-체인-탐색/)에 정리했다.

## JWT 시크릿 하드코딩

`application.yml`을 열어보니 이런 게 있었다.

```yaml
secret: ${JWT_SECRET:default-secret-key-for-development...}
```

환경변수가 없으면 기본값으로 돌아간다. 개발 편의를 위해 넣은 건데, 이게 프로덕션에서도 그대로 적용될 수 있다. 기본값을 제거하고 `${JWT_SECRET}`만 남겼다. 환경변수 미설정 시 서버가 아예 안 뜬다.

추가로 `@PostConstruct`에서 시크릿 길이를 검증하게 했다. 32바이트(256비트) 미만이면 서버 시작 실패. HS256 알고리즘도 명시적으로 지정했다. `.signWith(secretKey)` 같은 암묵적 선택은 `alg:none` 공격에 취약할 수 있다.

자세한 내용은 [JWT 보안 학습 포스트](/posts/JWT-보안과-Refresh-Token-Rotation/)에 정리했다.

## Refresh Token Rotation

기존 상태를 정리하면 이렇다.

- 로그아웃 엔드포인트: 없음
- Refresh Token 교체: 안 함
- 토큰 무효화: 불가능

Refresh Token을 한 번 탈취당하면 만료될 때까지 계속 쓸 수 있었다. 피해자가 할 수 있는 게 없다. 비밀번호를 바꿔도 기존 토큰은 유효하다.

해결책은 Refresh Token Rotation이다. 매번 토큰을 갱신할 때 새 RT를 발급하고, 이전 RT를 캐시(Redis)에 저장한다. 이미 사용된 RT가 다시 오면 탈취로 간주하고 해당 사용자의 모든 토큰을 무효화한다.

```
1. Client → /refresh (RT-1)
2. Server → RT-1 확인 → 새 AT + RT-2 발급, RT-1 무효화
3. 공격자 → /refresh (RT-1) → 이미 사용됨 → 전체 무효화
```

Logout 엔드포인트도 추가했다. `POST /api/v1/auth/logout`을 호출하면 캐시에서 RT를 삭제한다. SecurityConfig에서 이 경로는 인증 필수로 분리했다. 기존에 `/api/v1/auth/**`가 전부 permitAll이었는데, toss-login과 refresh만 permitAll로 두고 logout은 authenticated로 바꿨다.

[JWT 보안 학습 포스트](/posts/JWT-보안과-Refresh-Token-Rotation/)에 Rotation 구현 패턴을 자세히 정리했다.

## 보안 헤더 + CORS

SecurityConfig에 보안 헤더가 하나도 없었다.

```java
.headers(headers -> headers
    .frameOptions(frame -> frame.deny())
    .contentTypeOptions(Customizer.withDefaults())
    .httpStrictTransportSecurity(hsts -> hsts
        .includeSubDomains(true)
        .maxAgeInSeconds(31536000)))
```

`X-Frame-Options: DENY` — 클릭재킹 방어. 우리 페이지를 iframe으로 감싸서 사용자를 속이는 공격을 막는다. `HSTS` 31536000초(1년) — 브라우저에게 "이 사이트는 항상 HTTPS로 접속해"라고 알려준다.

CORS도 문제였다. `allowedHeaders`가 `*`(와일드카드)였다. 아무 헤더나 다 허용한다는 뜻이다. Authorization, Content-Type, Accept, X-Requested-With만 화이트리스트로 제한했다.

[보안 헤더 학습 포스트](/posts/보안-헤더와-Webhook-서명-검증/)에 각 헤더의 역할을 정리했다.

## Webhook HMAC-SHA256

PG 결제 콜백(`PaymentWebhookController`)이 `permitAll`이었다. 인증 없이 누구나 호출 가능. "결제가 완료됐다"는 가짜 요청을 보내면 서버가 그대로 처리한다. 공짜로 프리미엄 구독을 얻을 수 있었다는 뜻이다.

택배 배송 알림에 비유하면, 아무나 "배송 완료" 문자를 보낼 수 있는 상태. 진짜 택배 기사가 보낸 건지 확인할 방법이 없다.

HMAC-SHA256 서명 검증을 추가했다. PG사와 공유하는 시크릿 키로 요청 본문의 해시를 생성하고, `X-Webhook-Signature` 헤더에 담긴 서명과 비교한다. 서명이 일치하지 않으면 403을 반환한다.

Callback 엔드포인트도 마찬가지. 토스 계정 삭제 콜백에 `X-Callback-Secret` 검증을 추가했다. 기존에는 URL에 userKey만 들어오면 계정을 삭제했다. HIGH 취약점이 맞다.

[보안 헤더 학습 포스트](/posts/보안-헤더와-Webhook-서명-검증/)에 HMAC 서명 검증 구현을 정리했다.

## 테스트 리팩토링

보안 수정의 부작용이 테스트에서 터졌다. 서비스 메서드 시그니처가 전부 바뀌었으니 당연하다.

`updateRecipe(recipeId, title, category, description)` → `updateRecipe(recipeId, userId, title, category, description)`. 이런 식으로 userId 파라미터가 중간에 끼어들었다. 테스트 파일 8개에서 메서드 호출부를 수정했다.

| 테스트 파일 | 수정 내용 |
|-----------|----------|
| RecipeServiceTest | updateRecipe, deleteRecipe에 userId 추가 |
| RecipeControllerTest | mock 시그니처 수정 |
| RecipeVersionServiceTest | createVersion, updateVersion에 userId 추가 |
| PantryServiceTest | deleteItem에 userId 추가, UpdatePantryItemCommand에 userId 필드 추가 |
| CookingLogServiceTest | getLog에 userId 추가 |
| CookingLogControllerTest | mock 시그니처 수정 |
| PantryControllerTest | mock 시그니처 수정 |
| PantryConcurrencyIntegrationTest | UpdatePantryItemCommand 생성자 수정 |

[Fake Repository 학습 포스트](/posts/Fake-Repository-테스트-추상화/)에서 만든 테스트 인프라 덕분에 서비스 테스트 수정은 빨랐다. 컨트롤러 `@WebMvcTest`에서 mock 시그니처 맞추는 게 더 귀찮았다.

## Claude Code 협업

작업 순서가 중요했다. 보안은 하나 고치면 다른 데서 터지는 영역이다. 배관 공사하는데 한쪽 수도꼭지 잠그면 다른 쪽에서 물이 새는 것과 비슷하다.

순서를 이렇게 잡았다.

1. **IDOR** — 가장 먼저. 서비스 메서드 시그니처가 바뀌는 근본 변경
2. **JWT 시크릿 + 알고리즘** — 인증 인프라 수정
3. **Refresh Token Rotation + Logout** — 토큰 생명주기 추가
4. **보안 헤더 + CORS** — 인프라 설정
5. **Webhook/Callback 서명** — 외부 연동 보안
6. **테스트 수정** — 마지막. 위의 변경을 전부 반영

Claude Code에 보안 감사를 요청하면 취약점 목록이 쏟아진다. 여기까지는 좋다. 문제는 "어떤 순서로 고치느냐"다. IDOR을 나중에 고치면 테스트를 두 번 수정해야 한다. JWT 시크릿을 먼저 고치면 Rotation 작업할 때 충돌이 난다.

3개 에이전트를 병렬로 돌렸다.
- **에이전트 1**: IDOR 수정 (Controller + Service 시그니처 변경)
- **에이전트 2**: 보안 헤더 + CORS + JWT 시크릿 강화
- **에이전트 3**: Logout + Refresh Token Rotation + Webhook/Callback

에이전트 1이 끝난 뒤에 테스트 수정을 에이전트 3에 합류시켰다. 시그니처가 확정돼야 테스트를 고칠 수 있으니까.

빌드는 성공. 도메인 테스트도 통과. API 컨트롤러 테스트는 Spring Context 로딩 문제로 일부 실패했는데, 이건 기존에도 있던 문제라 보안 수정과는 무관하다.

## 배포 시 필요한 환경변수

| 환경변수 | 용도 | 비고 |
|---------|------|------|
| `JWT_SECRET` | JWT 서명 | 32바이트 이상 필수, 미설정 시 서버 시작 실패 |
| `TOSS_CALLBACK_SECRET` | 토스 콜백 인증 | 토스와 사전 공유 |
| `TOSS_WEBHOOK_SECRET` | PG 웹훅 서명 검증 | PG사와 사전 공유 |

FE도 수정이 필요하다. Refresh Token Rotation으로 인해 `/refresh` 응답에 새 RT가 포함된다. FE에서 RT 저장 로직을 업데이트해야 한다.

## 정리

| 항목 | Before | After |
|------|--------|-------|
| IDOR | 7개 엔드포인트 무방비 | 전체 소유권 검증 |
| JWT Secret | 하드코딩 기본값 | 환경변수 필수 + 길이 검증 |
| Refresh Token | 탈취 시 영구 사용 가능 | Rotation + 전체 무효화 |
| Logout | 없음 | RT 삭제 엔드포인트 |
| 보안 헤더 | 없음 | X-Frame-Options, HSTS |
| CORS | 와일드카드 | 화이트리스트 |
| Webhook | permitAll | HMAC-SHA256 서명 검증 |
| Callback | 무인증 | Secret 검증 |

기능 개발보다 보안이 더 오래 걸렸다. 기능은 "돌아가면 된다"지만, 보안은 "안 뚫리면 된다"다. 증명 방향이 반대다. 모든 경로를 막아야 하니까.
