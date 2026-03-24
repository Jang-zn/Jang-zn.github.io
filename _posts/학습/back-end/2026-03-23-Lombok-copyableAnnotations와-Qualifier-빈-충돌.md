---
title: "[학습] Lombok copyableAnnotations — @Qualifier가 사라지는 문제"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-23 16:00:00 +0900
categories: [학습, Backend]
tags: [학습, Lombok, Spring, Qualifier, DI, 빈 충돌]
render_with_liquid: false
---

## 빈이 2개인데 어떤 걸 넣어야 할지 모르겠다

[앱인토스-통합 Api](/posts/Swagger-테스트-FE연결/)에서 [Discord 모듈](/posts/Discord-Webhook과-포트-패턴/)을 추가한 뒤 앱이 안 떴다. `ExecutorService` 빈이 `parseWorkerExecutor`와 `discordExecutor` 2개가 되면서 `ParseWorker`에서 충돌.

```
Parameter 2 of constructor in ParseWorker required a single bean,
but 2 were found:
  - parseWorkerExecutor
  - applicationTaskExecutor
```

`@Qualifier("parseWorkerExecutor")`를 필드에 붙여놨는데 무시됐다. 왜?

## Lombok이 포스트잇을 떼어버린다

Lombok의 `@RequiredArgsConstructor`는 `final` 필드를 받는 생성자를 자동 생성한다. 근데 필드에 붙은 어노테이션을 생성자 파라미터에 복사하지 않는다. 서류 복사기가 원본에 붙은 포스트잇은 복사 안 하는 거다.

```java
// 내가 쓴 코드
@Component
@RequiredArgsConstructor
public class ParseWorker {
    @Qualifier("parseWorkerExecutor")
    private final ExecutorService parseWorkerExecutor;
}

// Lombok이 실제로 생성한 코드
public ParseWorker(ExecutorService parseWorkerExecutor) {
    // ← @Qualifier 없음!
    this.parseWorkerExecutor = parseWorkerExecutor;
}
```

Spring의 생성자 주입은 생성자 파라미터의 어노테이션을 기준으로 빈을 결정한다. 필드의 `@Qualifier`는 생성자 주입 시 참조되지 않는다. 필드명이 빈 이름과 같으면 우연히 동작할 수 있지만, 그건 Spring의 fallback 동작이지 `@Qualifier`가 동작한 게 아니다.

## 해결 — lombok.config

프로젝트 루트에 `lombok.config`를 만든다.

```properties
config.stopBubbling = true
lombok.copyableAnnotations += org.springframework.beans.factory.annotation.Qualifier
```

| 설정 | 의미 |
|------|------|
| `config.stopBubbling = true` | 상위 디렉토리 설정 상속 차단 |
| `copyableAnnotations += ...` | 필드 → 생성자 파라미터로 복사할 어노테이션 지정 |

이제 Lombok이 생성하는 코드에 `@Qualifier`가 포함된다.

```java
// 설정 후 Lombok이 생성한 코드
public ParseWorker(
    @Qualifier("parseWorkerExecutor")  // ← 복사됨
    ExecutorService parseWorkerExecutor
) { ... }
```

## Docker 빌드 함정

`lombok.config`는 컴파일 타임에 읽힌다. Dockerfile에서 COPY를 안 하면 로컬에서는 되는데 Docker 빌드에서만 터진다. 로컬 개발 환경에서 잘 되니까 "되는구나" 하고 넘어가면 배포할 때 터진다.

```dockerfile
COPY lombok.config .    # 반드시 필요
COPY gradle gradle
COPY build.gradle settings.gradle ./
```

실제로 이번에 Docker 빌드에서 빈 충돌이 나서 발견했다. 로컬에서는 IDE가 lombok.config를 알아서 읽어주니까 문제가 없었다.

## 다른 해결 방법들

| 방법 | 장점 | 단점 |
|------|------|------|
| `@Qualifier` + `copyableAnnotations` | 명시적, Lombok 호환 | `lombok.config` 필요 |
| `@Primary` | 설정 간단 | 기본 빈 하나만 지정 가능 |
| 필드명 = 빈 이름 | 추가 설정 없음 | 암묵적, 의도 불명확 |
| 직접 생성자 작성 | Lombok 의존 없음 | 보일러플레이트 증가 |

`@Primary`는 기본 빈을 하나만 지정할 수 있어서, 빈이 3개 이상이면 한계가 있다. 필드명 매칭은 암묵적이라 리팩토링할 때 깨지기 쉽다. `copyableAnnotations`가 명시적이면서 Lombok 편의성도 유지하는 조합이다.

[분산 락](/posts/분산-락/)에서 Redis 빈이 여러 개일 때도 같은 문제가 생길 수 있다. 같은 타입의 빈을 2개 이상 등록하면 이 설정을 먼저 챙기는 게 좋다.
