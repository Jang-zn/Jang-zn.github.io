---
title: "[학습] SpringDoc 인터페이스 분리 — 컨트롤러에서 문서를 떼어내기"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-23 12:00:00 +0900
categories: [학습, Backend]
tags: [학습, SpringDoc, OpenAPI, Swagger, Clean Architecture]
render_with_liquid: false
---

## 어노테이션이 로직을 덮어버린다

[앱인토스-통합 Api](/posts/Swagger-테스트-FE연결/)에서 35개 컨트롤러에 Swagger 문서를 달아야 했다. 컨트롤러에 직접 `@Operation`, `@ApiResponses`, `@Tag`를 붙이면 메서드 하나에 어노테이션이 6~8개씩 붙는다. 실제 비즈니스 로직은 맨 아래 한 줄인데, 위에 어노테이션 탑이 쌓인다.

코드 리뷰할 때도 문서 수정(응답 코드 추가 등)과 로직 수정이 한 diff에 섞인다. 어노테이션은 "직원의 업무 매뉴얼"이다. 매뉴얼을 직원 등에 테이프로 붙여놓으면 일을 못 한다. 매뉴얼은 서랍에 따로 두는 게 맞다.

## 인터페이스 분리 패턴

`controller/docs/` 패키지에 문서 전용 인터페이스를 만든다. 컨트롤러는 그 인터페이스를 `implements`만 하면 된다.

```
controller/
├── docs/
│   ├── AdminBannerControllerDocs.java
│   └── ...
├── AdminBannerController.java  // implements AdminBannerControllerDocs
└── ...
```

Docs 인터페이스에 `@Tag`, `@Operation`, `@ApiResponses`를 전부 몰아넣는다. 컨트롤러에는 `implements` 한 줄만 추가되고, Swagger 관련 어노테이션이 하나도 없다.

```java
// docs 인터페이스 — 문서 전담
@Tag(name = "배너 관리")
public interface AdminBannerControllerDocs {
    @Operation(summary = "배너 등록")
    @ApiResponses({
        @ApiResponse(responseCode = "200"),
        @ApiResponse(responseCode = "401"),
        @ApiResponse(responseCode = "403")
    })
    ApiResponse<BannerResponse> createBanner(...);
}

// 컨트롤러 — 로직 전담
@RestController
@RequiredArgsConstructor
public class AdminBannerController implements AdminBannerControllerDocs {
    // @Tag, @Operation, @ApiResponses 없음
    @PostMapping
    @PreAuthorize("hasAnyRole('ADMIN', 'SUPER_ADMIN')")
    public ApiResponse<BannerResponse> createBanner(...) {
        return ApiResponse.ok(bannerService.createBanner(...));
    }
}
```

## SpringDoc이 인터페이스 어노테이션을 인식하는 원리

표준 Java에서는 인터페이스의 메서드 어노테이션이 구현체로 상속되지 않는다. 하지만 SpringDoc은 내부적으로 구현 인터페이스까지 탐색한다. 컨트롤러에서 `@Operation`을 못 찾으면, `implements`한 인터페이스를 확인한다. 거기서 찾으면 그 메타데이터를 사용한다.

springdoc-openapi 고유의 동작이다. Spring MVC의 `@RequestMapping`이나 `@Valid` 같은 어노테이션은 이 방식이 안 통한다 — 그건 여전히 컨트롤러에 직접 붙여야 한다.

## 글로벌 보안 스키마

개별 메서드마다 `@SecurityRequirement`를 붙이는 것도 비효율적이다. `OpenApiConfig`에서 전역으로 Bearer JWT 스키마를 설정하면 Swagger UI에 Authorize 버튼이 생기고, 한 번 토큰을 입력하면 모든 API 요청에 자동 적용된다.

```java
@Bean
public OpenAPI openAPI() {
    return new OpenAPI()
        .addSecurityItem(new SecurityRequirement().addList("Bearer JWT"))
        .components(new Components()
            .addSecuritySchemes("Bearer JWT",
                new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")));
}
```

## 적용 결과

| 항목 | 수치 |
|------|------|
| 문서화된 컨트롤러 | 35개 (tastenote 17 + admin 18) |
| Docs 인터페이스 | 35개 |
| 컨트롤러 변경 | `implements` 한 줄 추가 |

[클린 아키텍처 인프라 추상화](/posts/클린-아키텍처-인프라-추상화/)에서 서비스와 인프라를 포트/어댑터로 분리한 것처럼, 컨트롤러와 문서도 인터페이스로 분리한다. 관심사 분리 원칙은 어디서든 써먹을 수 있다.
