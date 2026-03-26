---
title: "[학습] Docker Compose YAML Anchor — 복붙 지옥 탈출"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-23 10:00:00 +0900
categories: [학습, Backend]
tags: [학습, Docker, YAML, docker-compose, 인프라]
difficulty: ⭐3
render_with_liquid: false
---

## 복붙하다 빠뜨렸다

[앱인토스-통합 Api](/posts/Swagger-테스트-FE연결/)의 docker-compose에 10개 서비스가 있다. prod/dev API 4개, Redis 2개, MySQL, Nginx, Prometheus, Grafana. 이 중 API 서비스 4개가 거의 동일한 환경변수, 로깅, depends_on을 갖고 있었다.

환경변수 하나 추가하면 4곳을 수정해야 한다. 실제로 admin-api에 `CDN_BASE_URL`이 누락됐다. 복붙하다 하나 빼먹은 거다. dev 환경에서만 이미지 URL이 깨져서 찾는 데 시간이 걸렸다.

## YAML Anchor

YAML에는 "복사-붙여넣기 자동화기"가 내장돼 있다. `&이름`으로 원본을 정의하고 `*이름`으로 참조한다. 냉장고에 반찬 한 번 만들어두면 매 끼니마다 새로 만들 필요 없이 꺼내 쓰는 거다.

| 문법 | 역할 |
|------|------|
| `&이름` | Anchor 선언 — 원본 정의 |
| `*이름` | Alias 참조 — 원본 복사 |
| `<<: *이름` | Merge Key — 맵을 현재 맵에 병합 |

## 적용

docker-compose 최상위에 `x-` 접두사로 Extension Fields를 선언한다. `x-`로 시작하면 Docker Compose 엔진이 무시한다. 서비스로 해석하지 않는다. 순수하게 Anchor 저장소 역할만 한다.

```yaml
x-logging: &default-logging
  logging:
    options:
      max-size: "10m"
      max-file: "3"

x-common-env: &common-env
  DB_USERNAME: ${DB_USERNAME:-root}
  JWT_SECRET: ${JWT_SECRET}
  AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
  CDN_BASE_URL: ${CDN_BASE_URL:-}

x-depends-on-prod: &depends-on-prod
  mysql:
    condition: service_healthy
  redis-prod:
    condition: service_started
```

서비스에서 `<<: *앵커이름`으로 병합하고, 서비스 고유 값은 그 아래에 추가하면 된다. 같은 키가 있으면 서비스 값이 우선.

```yaml
services:
  tastenote-api:
    environment:
      <<: *common-env
      SPRING_PROFILES_ACTIVE: prod
      DB_URL: jdbc:mysql://mysql:3306/AIT-prod
    depends_on:
      <<: *depends-on-prod
    <<: *default-logging
```

## 오버라이드 규칙

Merge Key는 맵(딕셔너리)에서만 동작한다. 같은 키가 있으면 로컬 값이 Anchor 값을 덮어쓴다. 리모컨의 기본 설정을 불러온 다음, 내 방에 맞게 볼륨만 바꾸는 거다.

| 상황 | 동작 |
|------|------|
| Anchor에만 키가 있음 | 그대로 복사 |
| 서비스에만 키가 있음 | 그대로 유지 |
| 양쪽 모두 같은 키 | 서비스 값이 우선 |

## 주의사항

### 리스트는 병합 안 된다

`<<`는 맵 전용이다. 배열(volumes, ports 등)은 Merge Key로 합칠 수 없다. 통째로 참조(`*이름`)만 가능하고, 항목을 추가하거나 오버라이드하는 건 안 된다.

```yaml
# 이건 안 된다
x-volumes: &common-volumes
  - ./data:/app/data

services:
  my-api:
    volumes:
      <<: *common-volumes    # Error!
      - ./logs:/app/logs
```

배열을 재사용하려면 `*이름`으로 통째 참조만 가능하다. 추가 항목이 필요하면 Anchor 자체에 다 넣어야 한다.

### 다중 Anchor 병합

여러 Anchor를 한 번에 병합할 때는 리스트 형태로 쓴다. 키 충돌 시 첫 번째 Anchor가 우선.

```yaml
environment:
  <<:
    - *common-env
    - *extra-env    # 충돌 시 common-env가 우선
```

## 결과

265줄에서 218줄로 줄었다. 환경변수 하나 추가할 때 Anchor 한 곳만 수정하면 전체 서비스에 반영된다. `CDN_BASE_URL` 누락 같은 실수가 구조적으로 불가능해졌다.

작년 학습 과정에서 Docker Compose를 다룰 때 이 기능을 몰랐다. 서비스가 3개 이하면 복붙해도 괜찮다. 4개부터는 Anchor를 쓰는 게 맞다.
