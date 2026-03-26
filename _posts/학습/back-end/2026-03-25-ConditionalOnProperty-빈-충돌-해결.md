---
title: "[학습] @ConditionalOnProperty로 모듈간 빈 충돌 해결"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-25 17:00:00 +0900
categories: [학습, Backend]
tags: [학습, Spring, ConditionalOnProperty, 멀티모듈, 빈충돌]
difficulty: ⭐5
render_with_liquid: false
---

## 쓰지도 않는 빈이 왜 로딩되냐

[TasteNote BE](/posts/인앱결제-광고-제거-시스템/)는 멀티모듈 프로젝트다. admin-api에서 `TossOrderVerificationAdapter`를 쓰려면 `infra-security` 모듈을 의존해야 한다. 문제는 infra-security에 JWT, Security 관련 빈이 잔뜩 있다는 것. admin-api는 이런 빈이 필요 없다.

냉장고에서 버터만 꺼내려는데 냉장고 전체가 딸려 나오는 상황.

## 시도 1: @ComponentScan exclude — 실패

```java
@ComponentScan(excludeFilters = @Filter(type = REGEX, pattern = ".*Jwt.*"))
```

이론상 맞는데, `@WebMvcTest` 같은 테스트 슬라이스가 자체적으로 ComponentScan을 돌려서 exclude가 무시된다. 테스트에서만 터지는 유령 버그.

## 시도 2: @ConditionalOnProperty — 성공

JWT 관련 빈에 조건을 건다.

```java
@Component
@ConditionalOnProperty(name = "app.jwt.secret")
public class JwtTokenProvider { ... }
```

`app.jwt.secret` 프로퍼티가 있을 때만 빈이 등록된다. tastenote-api는 `application.yml`에 이 값이 있으니 JWT 빈이 뜨고, admin-api는 이 값이 없으니 JWT 빈이 안 뜬다.

수도꼭지에 조건을 거는 거다. "이 방에 세면대가 있을 때만 물을 틀어라."

## 왜 모듈을 쪼개지 않았나

"infra-security에서 Toss 어댑터를 빼서 infra-toss 모듈로 분리하면 되잖아" — 맞다. 하지만 1인 프로젝트에서 모듈 수를 늘리는 건 관리 비용이 늘어나는 거다. 어댑터 하나 때문에 모듈을 만들면 build.gradle 하나, 패키지 구조 하나, 의존성 관리 포인트 하나가 늘어난다.

`@ConditionalOnProperty`로 빈 로딩을 제어하면 모듈을 쪼개지 않고도 해결된다. 규모가 커지면 그때 분리하면 된다.

## @Conditional 계열 정리

| 어노테이션 | 조건 |
|-----------|------|
| `@ConditionalOnProperty` | 프로퍼티 존재/값 매칭 |
| `@ConditionalOnBean` | 특정 빈이 존재할 때 |
| `@ConditionalOnMissingBean` | 특정 빈이 없을 때 |
| `@ConditionalOnClass` | 특정 클래스가 클래스패스에 있을 때 |
| `@Profile` | 프로필 매칭 |

`@Profile("tastenote")`도 비슷한 효과를 낼 수 있지만, `@ConditionalOnProperty`가 더 명시적이다. "이 빈은 JWT 설정이 있을 때만 필요하다"는 의도가 코드에 드러난다.

## 프로젝트 적용

```
infra-security 모듈:
├── JwtTokenProvider          → @ConditionalOnProperty("app.jwt.secret")
├── JwtAuthenticationFilter   → @ConditionalOnProperty("app.jwt.secret")
├── SecurityConfig            → @ConditionalOnProperty("app.jwt.secret")
└── TossOrderVerificationAdapter → 조건 없음 (항상 로드)
```

admin-api가 infra-security를 의존해도 JWT 빈이 안 뜬다. `TossOrderVerificationAdapter`만 깔끔하게 가져다 쓸 수 있다.
