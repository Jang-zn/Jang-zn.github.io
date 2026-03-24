---
title: "[TasteNote] Swagger 문서화, 테스트 81개, FE-BE 연결"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-23
categories: [프로젝트, TasteNote]
tags: [프로젝트, TasteNote, Swagger, SpringDoc, WebMvcTest, Docker, Vite]
render_with_liquid: false
---

## 세 가지

[피드백 기능](/posts/피드백-기능과-메인-컬러-변경/) 끝내고 3일 쉬었다. 오늘은 BE 쪽 정리 작업. PR 2개 머지. 그리고 FE에서 실제 BE 서버에 연결하는 준비.

1. Swagger 문서화 + Docker 정리 + 버그 수정 (PR #17)
2. 컨트롤러 테스트 81개 (PR #18)
3. FE — 실제 BE 서버 연결 준비

## PR #17 — Swagger 문서화 + 인프라 정리

커밋 8개. 내용이 뒤섞여 있지만 흐름은 이렇다.

### Swagger 문서화 — 인터페이스 분리

tastenote-api 17개, admin-api 18개 컨트롤러에 Swagger 문서가 없었다. 컨트롤러에 직접 `@Operation`, `@ApiResponses`를 붙이면 어노테이션이 비즈니스 로직을 덮어버린다. 메서드 하나에 어노테이션 6~8개. 코드 리뷰할 때 문서 변경과 로직 변경이 한 diff에 섞인다.

인터페이스 분리 패턴으로 해결했다. `controller/docs/` 패키지에 문서 전용 인터페이스를 만들고, 컨트롤러가 `implements`만 하면 된다. 컨트롤러에는 `implements XxxControllerDocs` 한 줄만 추가. 깔끔하다. 자세한 건 [학습 포스트](/posts/SpringDoc-인터페이스-분리-패턴/)에 정리했다.

401/403 응답 코드도 추가. 인증 필요 엔드포인트에 401, `@PreAuthorize` 적용 메서드에 403.

### .gitignore 충돌

문서 인터페이스가 `controller/docs/` 패키지에 있는데, `.gitignore`에 `docs/`가 있어서 git이 추적을 안 했다. 루트 `docs/` 디렉토리만 무시하려던 건데 모든 하위 경로의 `docs/`까지 잡아버린 거다. `/docs/`로 수정해서 루트만 한정.

### ParseWorker 빈 충돌

Discord 모듈을 추가하면서 `ExecutorService` 빈이 2개가 됐다. `parseWorkerExecutor`와 `discordExecutor`. Lombok의 `@RequiredArgsConstructor`가 필드의 `@Qualifier`를 생성자 파라미터에 복사하지 않아서 Spring이 어떤 빈을 주입할지 모르겠다고 에러를 뱉었다.

`lombok.config`에 `copyableAnnotations` 설정을 추가해서 해결. Dockerfile에도 `lombok.config` COPY를 추가해야 한다. 안 하면 로컬에서는 되는데 Docker 빌드에서만 터진다. 자세한 건 [학습 포스트](/posts/Lombok-copyableAnnotations와-Qualifier-빈-충돌/)에.

### Docker 환경변수 보완

홈서버 배포 시 AWS 자격증명이 컨테이너에 안 들어가서 S3Client 생성 실패. admin-api에 `app.storage` 설정 블록도 누락돼 있었다. `.env.example`에 AWS 키 항목 추가하고, docker-compose에 환경변수 보완.

### docker-compose YAML Anchor

10개 서비스에 로깅, 환경변수, depends_on이 반복되고 있었다. 환경변수 하나 추가하면 4곳을 수정해야 한다. 실제로 admin-api에 `CDN_BASE_URL` 누락이 발생했다. 복붙하다 하나 빠뜨린 거다.

YAML Anchor로 해결. `x-common-env`, `x-logging`, `x-depends-on-prod/dev`를 선언하고 `<<: *앵커이름`으로 병합. 265줄에서 218줄로 줄었다. 자세한 건 [학습 포스트](/posts/Docker-Compose-YAML-Anchor/)에.

## PR #18 — 컨트롤러 테스트 81개

커밋 4개. 테스트 인프라 구성 → 테스트 작성 순서.

### 테스트 인프라

`@WebMvcTest`로 컨트롤러만 띄우는데, Spring Security가 개입하면 단순하지 않다. JWT 필터가 요청을 가로채서 토큰 없으면 무조건 401. `@PreAuthorize`는 `@EnableMethodSecurity` 없으면 무시된다.

테스트 전용 `TestMethodSecurityConfig`를 만들었다. JWT 필터를 빼고 메서드 레벨 보안만 활성화. `AdminControllerTestSupport` 베이스 클래스에 MockMvc, Mock 빈, 역할별 Principal 상수를 모았다. 16개 테스트 클래스에서 각각 15줄의 보일러플레이트가 사라졌다. 자세한 건 [학습 포스트](/posts/WebMvcTest와-Spring-Security-테스트-전략/)에.

### 테스트 내용

| 범위 | 테스트 수 | 내용 |
|------|----------|------|
| admin-api 인증/권한 | 22개 | 401 미인증, 403 권한 부족, SUPER_ADMIN 전용 |
| admin-api 권한+404 | 50개 | 11개 컨트롤러의 404 미존재 + VIEWER/ADMIN 403 |
| tastenote-api 404/409 | 9개 | 미존재 404, 중복 409, 필수 필드 누락 400 |

총 81개. Claude Code에 "admin-api 전체 컨트롤러에 인증/권한/404 테스트 추가해"라고 지시했다. 테스트 인프라부터 만들고, 컨트롤러별로 커밋을 나눠서 진행.

403 테스트에서 주의할 점이 있다. `@Valid` 검증이 `@PreAuthorize`보다 먼저 실행된다. 잘못된 요청 본문으로 403 테스트를 하면 403이 아니라 400이 나온다. Claude Code가 처음에 이걸 놓쳐서 한 번 수정했다.

## FE — 실제 BE 연결 준비

### Vite 다운그레이드

`@apps-in-toss/web-framework@2.0.5`가 Vite 6~7을 기대한다. Vite 8에서 `pluginHooks` API가 바뀌어서 `granite dev` 실행이 안 됐다. Vite 8 → 7, `@vitejs/plugin-react` 6 → 4로 다운그레이드.

### MSW 토글

지금까지 MSW로 API를 모킹해서 개발했다. [BE가 홈서버에 배포됐으니](/posts/홈서버-배포와-CI-CD-구축/) 실제 서버에 연결할 차례. MSW 활성화 조건에 `VITE_ENABLE_MSW` 환경변수 체크를 추가했다. `VITE_ENABLE_MSW=false`로 실행하면 MSW 없이 실제 BE에 연결된다. 기본값은 기존과 동일하게 MSW 활성화.

Vite proxy target 포트도 8080 → 8180으로 변경. dev API 서버 포트에 맞춤.

## Claude Code 협업

오늘은 거의 Claude Code한테 맡겼다. Swagger 문서화는 35개 컨트롤러에 인터페이스 생성 + 응답 코드 추가라서 수동으로 하면 하루종일이다. "인터페이스 분리 패턴으로 전체 컨트롤러 Swagger 문서화해"라고 지시. 컨트롤러별로 커밋을 나눠서 진행해줬다.

테스트도 마찬가지. "admin-api 전체 컨트롤러에 401/403/404 테스트 추가"로 81개 테스트가 나왔다. 테스트 인프라(베이스 클래스, Security 설정)부터 만들고 컨트롤러별로 커밋을 나누는 구조도 좋았다.

docker-compose YAML Anchor 정리도 "중복 제거해"라고 했더니 Extension Fields 패턴을 적용. 다만 `CDN_BASE_URL` 누락은 내가 먼저 발견해서 "이것도 추가해"라고 따로 지시.

## 오늘의 결과

| 항목 | 상태 |
|------|------|
| Swagger 문서화 (35개 컨트롤러) | 완료 |
| 401/403 응답 코드 추가 | 완료 |
| docker-compose YAML Anchor 정리 | 완료 |
| ParseWorker 빈 충돌 해결 | 완료 |
| Docker 환경변수 보완 | 완료 |
| 컨트롤러 테스트 81개 | 완료 |
| Vite 다운그레이드 | 완료 |
| MSW 토글 + BE 연결 준비 | 완료 |
| PR #17, #18 머지 | 완료 |

BE 정리가 끝났다. 다음은 FE에서 MSW를 끄고 실제 API로 연결하면서 통합 테스트를 진행할 차례.
