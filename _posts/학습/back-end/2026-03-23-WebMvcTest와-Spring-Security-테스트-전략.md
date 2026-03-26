---
title: "[학습] @WebMvcTest와 Spring Security 테스트 전략"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-23 14:00:00 +0900
categories: [학습, Backend]
tags: [학습, WebMvcTest, Spring Security, 테스트, PreAuthorize]
difficulty: ⭐5
render_with_liquid: false
---

## 컨트롤러만 테스트하고 싶다

[앱인토스-통합 Api](/posts/Swagger-테스트-FE연결/)에서 admin-api와 tastenote-api에 컨트롤러 테스트를 추가했다. 총 81개. `@WebMvcTest`로 컨트롤러 레이어만 띄우는 슬라이스 테스트다.

`@SpringBootTest`가 "건물 전체를 지어서 테스트"라면, `@WebMvcTest`는 "현관문만 만들어서 문 여닫기를 테스트"하는 거다. DB, Redis, 외부 서비스 로드 없이 빠르게 돌린다.

근데 Spring Security가 개입하면 단순하지 않다.

## 세 가지 문제

### 1. JWT 필터가 모든 요청을 401로 만든다

프로덕션 SecurityConfig에 JWT 필터가 있다. 테스트에서 토큰 없이 요청하면 무조건 401. 실제로 테스트하고 싶은 건 컨트롤러 로직인데, 문 앞에서 경비원이 막는다.

### 2. @PreAuthorize가 무시된다

`@EnableMethodSecurity`가 없으면 `@PreAuthorize`가 동작하지 않는다. VIEWER로 요청해도 403이 아닌 200이 나온다. 권한 테스트가 의미 없어진다.

### 3. @AuthenticationPrincipal이 null

Security 설정 없이 테스트하면 principal이 null로 주입된다. NullPointerException.

## 해결 — 테스트 전용 Security 설정

JWT 필터를 빼고, 메서드 레벨 보안만 활성화하는 `TestMethodSecurityConfig`를 만들었다. 경비원은 해고하고, 방 문마다 출입 카드 리더기만 남기는 거다.

```java
@TestConfiguration
@EnableWebSecurity
@EnableMethodSecurity(proxyTargetClass = true)
public class TestMethodSecurityConfig {
    @Bean
    public SecurityFilterChain testSecurityFilterChain(HttpSecurity http) {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/admin/v1/auth/login").permitAll()
                .anyRequest().authenticated())
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) ->
                    res.setStatus(401))
                .accessDeniedHandler((req, res, e) ->
                    res.setStatus(403)));
        return http.build();
    }
}
```

### proxyTargetClass = true

이게 좀 까다로웠다. 컨트롤러가 [Docs 인터페이스](/posts/SpringDoc-인터페이스-분리-패턴/)를 `implements`하고 있다. Spring AOP 기본 동작은 JDK Dynamic Proxy(인터페이스 기반)다. 이 경우 `@PreAuthorize`가 컨트롤러 클래스에 있고 인터페이스에는 없어서 프록시가 어노테이션을 못 찾는다. `proxyTargetClass = true`로 CGLIB 프록시를 강제하면 클래스 기반 프록시가 생성되어 정상 동작한다.

## 베이스 클래스

16개 테스트 클래스에서 반복되는 설정을 `AdminControllerTestSupport`로 추출했다.

```java
@Import(TestMethodSecurityConfig.class)
abstract class AdminControllerTestSupport {
    @Autowired
    protected MockMvc mockMvc;

    @MockitoBean
    protected AdminJwtTokenProvider adminJwtTokenProvider;

    protected static final AdminPrincipal SUPER_ADMIN = ...;
    protected static final AdminPrincipal ADMIN = ...;
    protected static final AdminPrincipal VIEWER = ...;
}
```

각 테스트 클래스는 `extends AdminControllerTestSupport`만 하면 된다. Service만 `@MockitoBean`으로 추가.

## 테스트 패턴

`.with(user(principal))`로 가짜 인증 정보를 주입한다. JWT 토큰 파싱 없이 바로 인증된 사용자로 요청할 수 있다.

```java
// 401 — 인증 없이
mockMvc.perform(get("/api/admin/v1/banners"))
    .andExpect(status().isUnauthorized());

// 403 — VIEWER로 ADMIN 전용 API 호출
mockMvc.perform(post("/api/admin/v1/banners")
        .with(user(VIEWER))
        .contentType(APPLICATION_JSON)
        .content("{...}"))
    .andExpect(status().isForbidden());

// 404 — 존재하지 않는 리소스
given(bannerService.getBanner("x"))
    .willThrow(new BusinessException(ErrorCode.ENTITY_NOT_FOUND));
mockMvc.perform(get("/api/admin/v1/banners/x")
        .with(user(VIEWER)))
    .andExpect(status().isNotFound());
```

## 검증 순서 함정

HTTP 요청이 컨트롤러에 도달하기까지의 순서:

**인증(401) → @Valid(400) → @PreAuthorize(403) → 컨트롤러 로직(404/409/200)**

`@Valid` 검증이 `@PreAuthorize`보다 먼저 실행된다. 403 테스트에서 잘못된 요청 본문을 보내면 403이 아니라 400이 나온다. 403 테스트에서는 반드시 유효한 요청 본문을 보내야 한다. 처음에 이걸 놓쳐서 테스트가 깨졌다.

## 결과

| 항목 | 수치 |
|------|------|
| admin-api 테스트 | 72개 |
| tastenote-api 테스트 | 9개 |
| 테스트 인프라 클래스 | 2개 (Config + Support) |
| 보일러플레이트 절감 | 클래스당 ~15줄 |

작년 멘토링에서 "테스트는 인프라를 먼저 깔아라"라는 조언을 들었다. 테스트 하나 만드는 데 30줄의 설정이 필요하면 아무도 테스트를 안 쓴다. 베이스 클래스로 설정을 숨기면 테스트 작성 허들이 낮아진다. 이번에 체감했다.
