---
title: "[앱인토스-통합 Api] Jackson 전략 변경과 mTLS 전환"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-19
categories: [프로젝트, 앱인토스-통합 Api]
tags: [프로젝트, 앱인토스-통합 Api, Jackson, mTLS, Redis, 캐싱]
render_with_liquid: false
---

## 세 프로젝트 동시 작업

오늘은 BE, Admin FE, TasteNote FE 세 프로젝트를 동시에 건드렸다. 핵심은 하나. JSON 응답을 snake_case에서 camelCase로 통일하는 것.

[어제](/posts/분산락과-어드민-API-완성/) BE가 `PropertyNamingStrategy.SNAKE_CASE`로 설정되어 있어서 [Admin FE](/posts/API-클라이언트-인프라-구축/)에서 Axios 인터셉터로 snake_case ↔ camelCase 변환을 하고 있었다. [TasteNote](/posts/약관과-Firebase-수정/)에서도 Raw 타입과 mapper에서 snake_case 키를 camelCase로 매핑하고 있었다. 프론트엔드 2곳에서 같은 변환을 중복으로 하고 있는 거다. 서버에서 camelCase로 내려주면 이 변환 로직이 전부 필요 없어진다.

## Jackson SNAKE_CASE 제거

`application.yml`에서 `property-naming-strategy: SNAKE_CASE` 설정을 제거했다. `tastenote-api`와 `admin-api` 모듈 양쪽 다.

그리고 `JacksonConfig.java`를 삭제했다. 이 클래스가 `@Bean @Primary`로 ObjectMapper를 직접 생성하고 있었는데, 이러면 Spring Boot의 자동설정(`JacksonAutoConfiguration`)이 통째로 꺼진다. JavaTimeModule 자동 등록, yml 설정 적용, springdoc이나 jjwt-jackson 같은 라이브러리 커스터마이저 — 전부 무시된다. 자세한 건 [학습 포스트](/posts/Jackson-네이밍-전략-변경/)에 정리했다.

## 테스트 JSON 키 변경

API Breaking Change니까 테스트가 전부 깨진다. `authorization_code` → `authorizationCode`, `refresh_token` → `refreshToken`, `main_img_url` → `mainImgUrl`. 테스트 3개(Auth, Image, Recipe) 수정.

`mainImgUrl` 쪽이 실제로 위험했다. 네이밍 전략 없이는 `main_img_url`로 들어오는 요청이 `mainImgUrl` 필드에 바인딩되지 않고 조용히 `null`이 된다. 테스트에서 실제 JSON 키를 쓰지 않았으면 프로덕션에서 이미지 URL이 사라지는 버그가 됐을 것.

## Redis 캐시 키 버전닝

Jackson 전략이 바뀌면 Redis에 이미 저장된 캐시도 깨진다. 구 캐시는 snake_case 키로 직렬화되어 있는데 새 ObjectMapper는 camelCase를 기대한다. 역직렬화 실패 → 캐시 미스 → DB 직접 조회. [학습 포스트](/posts/Redis-캐시-직렬화-충돌/)에 정리.

캐시 키 프리픽스에 `v2`를 붙여서 해결했다. `recipe:{id}` → `recipe:v2:{id}`. TTL이 10분이니까 구 캐시는 자연 소멸. 이미 [캐시 스탬피드 방지](/posts/캐시-스탬피드/)가 적용되어 있어서 cold start도 안전하다.

## 토스 API 인증 mTLS 전환

토스 개발자 문서를 다시 읽어봤더니 서버 간 API 통신은 Basic Auth가 아니라 mTLS(상호 TLS 인증서)로 해야 한다. 기존 구현이 틀렸던 것.

`TossProperties`에서 `clientId`/`clientSecret`을 제거하고 `certPath`/`keyPath`로 교체. `TossPartnerApiClient`에서 PEM 인증서를 파싱해서 SSLContext를 구성하고 RestClient에 적용. 인증서가 설정되지 않은 로컬 개발 환경에서는 기본 HTTPS로 폴백되게 했다.

## Claude Code 협업

Jackson 전략 변경은 내가 "snake_case 전략 빼자, 프론트엔드 두 곳에서 변환하는 게 낭비다"라고 방향을 잡았다. Claude한테 실행을 맡겼는데, `JacksonConfig` 삭제 판단도 Claude가 자동설정 충돌을 분석해서 제안한 거다. "이 빈이 있으면 Spring Boot 자동설정이 꺼진다"는 분석이 정확했다.

Redis 캐시 충돌은 Claude가 먼저 잡았다. "네이밍 전략 바꾸면 기존 캐시 역직렬화가 실패합니다, 키 버전닝 하겠습니다"라고 선제적으로 처리. 이건 내가 놓칠 뻔한 부분이다.

mTLS 전환도 Claude가 토스 문서를 참조해서 "Basic Auth가 아니라 mTLS입니다"라고 알려줬다. 문서를 꼼꼼히 읽는 건 Claude가 낫다.

## 오늘의 결과

| 항목 | 상태 |
|------|------|
| Jackson SNAKE_CASE 전략 제거 | 완료 |
| JacksonConfig 삭제 (자동설정 복원) | 완료 |
| 테스트 JSON 키 camelCase 변환 | 완료 |
| Redis 캐시 키 v2 버전닝 | 완료 |
| 토스 API mTLS 전환 | 완료 |
| PR #11, #12 머지 | 완료 |

BE 진행률 **75~80%** → **80~85%**. API 응답 포맷이 통일되면서 프론트엔드 연동이 한결 깔끔해졌다.
