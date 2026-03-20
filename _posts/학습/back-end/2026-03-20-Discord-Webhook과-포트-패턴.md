---
title: "[학습] Discord Webhook과 Clean Architecture 포트 패턴"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-20 12:00:00 +0900
categories: [학습, Backend]
tags: [학습, Discord, Webhook, Clean Architecture, 비동기]
render_with_liquid: false
---

## DB 없이 CS를 운영한다

[앱인토스-통합 Api](/posts/홈서버-배포와-CI-CD-구축/)에서 사용자 피드백과 서버 에러 알림을 Discord Webhook으로 보내기로 했다. DB에 저장하지 않는다. 초기 단계에서 피드백 관리 시스템을 만드는 건 과하다. Discord 채널이 그 역할을 대신한다. 알림, 검색, 스레드, 팀 협업 — 다 된다.

피드백이 월 100건을 넘거나 SLA 추적이 필요해지면 그때 DB 기반 시스템을 만들어도 늦지 않다.

## Clean Architecture — 포트 패턴

Discord 연동을 `core-domain`에 직접 의존시키면 안 된다. Discord가 Slack으로 바뀌거나 DB 저장 방식으로 전환할 때 도메인 코드를 건드려야 하니까.

```
core-domain (포트 정의)
  └── DiscordNotificationPort: sendFeedback(), sendAlert(), sendReport()

infra-discord (어댑터 구현)
  └── DiscordWebhookAdapter implements DiscordNotificationPort
```

포트는 인터페이스다. 도메인은 "알림을 보낸다"는 것만 알고, 어떻게 보내는지는 모른다. [클린 아키텍처 인프라 추상화](/posts/클린-아키텍처-인프라-추상화/)에서 Redis, JPA에 적용한 것과 같은 패턴이다.

알림 데이터도 VO(Value Object)로 분리했다. `FeedbackNotification`, `AlertNotification`, `ReportNotification` — 각각 필요한 필드만 가진다.

## @Async와 Virtual Thread

알림 전송이 API 응답을 블로킹하면 안 된다. Discord API가 느려도 사용자 응답에 영향을 주면 안 된다.

`@Async`로 비동기 실행하되, 스레드 풀은 Virtual Thread 기반으로 구성했다. Java 21의 가상 스레드는 플랫폼 스레드보다 훨씬 가볍다. Discord 알림처럼 I/O 대기가 긴 작업에 적합하다.

## ApplicationEvent로 모듈 분리

서버 에러 알림을 보내려면 `GlobalExceptionHandler`에서 Discord 포트를 호출해야 한다. 근데 예외 처리기에 알림 로직이 직접 들어가면 결합도가 높아진다.

`ServerErrorEvent`를 발행하고, 별도의 `ServerErrorEventListener`가 비동기로 받아서 Discord로 보낸다. 예외 처리기는 이벤트만 발행하고 끝. 알림 실패가 예외 응답에 영향을 주지 않도록 try-catch로 감싼다.

## Rate Limiting

Discord API는 분당 30건 제한이 있다. 서버 에러가 폭주하면 rate limit에 걸린다. 인메모리 슬라이딩 윈도우로 webhook별 30 req/min 제한을 걸었다. 초과하면 해당 알림은 버린다. 알림을 위해 서비스가 느려지면 본말전도다.

## 입력 검증 2단계

사용자가 보내는 피드백과 Discord Embed 필드에 각각 검증이 필요하다.

| 단계 | 대상 | 제한 |
|------|------|------|
| API 입력 | category | 50자 |
| API 입력 | title | 200자 |
| API 입력 | description | 2000자 |
| Embed 필드 | SHORT (name 등) | 256자 truncate |
| Embed 필드 | LONG (description 등) | 1024자 truncate |

API 레벨에서 거부하는 것과 Embed 레벨에서 잘라내는 것은 다르다. API 검증은 사용자에게 에러를 돌려주고, Embed 절단은 Discord가 에러를 뱉지 않게 하는 안전장치다.

## Webhook URL 로그 마스킹

webhook URL에는 토큰이 포함되어 있다. 로그에 그대로 찍으면 보안 문제다. 토큰의 앞 4자만 남기고 나머지를 `***`로 마스킹한다.

## 정리

Discord Webhook은 초기 단계 CS 운영에 적합한 선택이다. Clean Architecture 포트 패턴으로 감싸두면 나중에 Slack이든 DB든 교체가 쉽다. 중요한 건 "알림 인프라가 서비스 응답에 영향을 주지 않는다"는 보장이다. @Async, try-catch, rate limiting — 전부 이 원칙을 지키기 위한 장치다.
