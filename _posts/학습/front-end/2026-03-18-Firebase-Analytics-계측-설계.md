---
title: "[학습] Firebase Analytics 계측 — 이벤트 설계에서 삽질까지"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-18 20:30:00 +0900
categories: [학습, Frontend]
tags: [학습, Firebase, Analytics, 이벤트 추적]
render_with_liquid: false
---

## 왜 필요했나

[TasteNote](/posts/약관과-Firebase-수정/)에 Firebase Analytics를 넣었다. 사용자가 어떤 페이지를 보고, 어떤 기능을 쓰고, 어디서 에러가 나는지 추적해야 앱을 개선할 수 있으니까. 근데 넣고 나서 버그가 5개 터졌다.

## Firebase Analytics가 뭔지

Google의 앱/웹 분석 도구다. 사용자 행동을 **이벤트** 단위로 수집한다. 이벤트에는 이름과 파라미터가 있다.

```typescript
logEvent(analytics, 'page_view', {
  page_title: '레시피 상세',
  page_location: '/recipes/abc-123',
});
```

Google Analytics 콘솔에서 이벤트별 횟수, 사용자 수, 전환율 같은 걸 볼 수 있다.

## 삽질 5가지

### 1. 초기화 전 이벤트 유실

Firebase SDK 초기화는 비동기다. 앱이 뜨자마자 `page_view` 이벤트가 발생하는데, SDK가 아직 준비 안 됐으면 이벤트가 그냥 사라진다. 첫 번째 `page_view`가 수집 안 되는 거다.

**해결: 이벤트 버퍼링.** 가게가 아직 안 열었는데 손님이 와버린 상황이다. 문 앞에 대기 줄을 세우고, 오픈하면 순서대로 들여보내는 거다. 초기화 완료 전에 들어오는 이벤트를 배열에 담아두고, 초기화 끝나면 순서대로 flush한다.

```typescript
let buffer: PendingEvent[] = [];
let initialized = false;

function trackEvent(name: string, params: Record<string, unknown>) {
  if (!initialized) {
    buffer.push({ name, params });
    return;
  }
  logEvent(analytics, name, params);
}

async function initialize() {
  await initializeApp(config);
  initialized = true;
  buffer.forEach(e => logEvent(analytics, e.name, e.params));
  buffer = [];
}
```

BE에서 메시지 큐를 쓰는 것과 비슷한 패턴이다. 처리할 준비가 안 됐으면 버퍼에 넣고 나중에 처리.

### 2. 이벤트 파라미터에 객체가 들어감

`api_error` 이벤트의 `error_type` 필드에 `{code, message}` 객체가 통째로 들어가고 있었다. Firebase Analytics의 이벤트 파라미터는 **문자열이나 숫자만** 받는다. 객체를 넣으면 `[object Object]`로 직렬화되어서 쓸모없는 데이터가 쌓인다.

```typescript
// before — 객체가 들어감
trackEvent('api_error', {
  error_type: err.response.data.error, // { code: "DUPLICATE", message: "..." }
});

// after — 문자열만 전달
trackEvent('api_error', {
  error_type: err.response.data.error.code, // "DUPLICATE"
});
```

**이벤트 파라미터 타입을 interface로 정의**해두면 이런 실수를 컴파일 타임에 잡을 수 있다.

```typescript
interface ApiErrorDetail {
  error_type: string;  // 여기서 string이 아니면 컴파일 에러
  endpoint: string;
  status_code: number;
}
```

### 3. 민감 정보가 이벤트에 포함됨

URL 검증 실패 이벤트에서 **사용자가 입력한 전체 URL**을 전송하고 있었다. URL에 토큰, 세션 ID, 개인 식별자 같은 게 포함될 수 있다. Analytics 데이터는 Google 서버에 저장되니까 민감 정보가 들어가면 안 된다.

```typescript
// before — 전체 URL 전송
trackEvent('url_validation_failed', { url: userInput });

// after — 도메인만 전송
trackEvent('url_validation_failed', {
  url_domain: new URL(userInput).hostname,
});
```

원칙: **분석에 필요한 최소한의 정보만 전송한다.** "어떤 사이트의 URL이 실패했는지"만 알면 되지, 정확한 URL은 필요 없다.

### 4. .env.production이 git에 올라감

Firebase 설정값(API key, project ID 등)이 담긴 `.env.production` 파일이 git에 커밋되어 있었다. 플레이스홀더 값이었지만, 이 파일이 빌드에 포함되면 프로덕션 Firebase 초기화가 실패한다.

```bash
git rm --cached .env.production
```

`.env.production.example` 템플릿을 만들고, 실제 값은 CI/CD 환경변수로 주입한다. 추가로 `isFirebaseConfigValid()` 함수로 플레이스홀더 값이 남아있는지 런타임에 검증한다.

```typescript
function isFirebaseConfigValid(config: FirebaseConfig): boolean {
  return Object.values(config).every(
    v => v && !v.startsWith('your-') && v !== 'placeholder'
  );
}
```

### 5. Node 엔진 버전 미명시

Firebase SDK가 Node 20+를 요구하는데 `package.json`에 `engines` 필드가 없었다. 다른 개발자(혹은 CI)가 Node 18로 빌드하면 알 수 없는 에러가 난다.

```json
{
  "engines": { "node": ">=20" }
}
```

`.npmrc`에 `engine-strict=true`까지 추가해서 호환 안 되는 환경에서 `npm install` 자체를 막는다.

## 이벤트 설계 원칙

삽질하면서 정리한 원칙들.

| 원칙 | 설명 |
|------|------|
| 파라미터는 원시 타입만 | 문자열, 숫자. 객체/배열 금지 |
| 민감 정보 제거 | URL, 토큰, 사용자 입력값 그대로 보내지 않기 |
| 초기화 전 버퍼링 | SDK 준비 전 이벤트 유실 방지 |
| 타입으로 계약 | 이벤트 파라미터를 interface로 정의 |
| 설정 검증 | 플레이스홀더/누락 값 런타임 체크 |

## BE 모니터링과 비교

BE에서 [Prometheus + Grafana로 모니터링](/posts/분산락과-어드민-API-완성/)을 세팅한 것과 같은 맥락이다.

| | FE (Firebase Analytics) | BE (Prometheus + Grafana) |
|---|---|---|
| 수집 대상 | 사용자 행동 (클릭, 페이지 뷰) | 서버 메트릭 (요청량, 지연시간) |
| 수집 방식 | 이벤트 push | 메트릭 pull (scraping) |
| 저장소 | Google Analytics | Prometheus TSDB |
| 시각화 | GA 콘솔 | Grafana 대시보드 |
| 커스텀 지표 | 커스텀 이벤트 | Micrometer counter/gauge |

둘 다 **"측정 안 하면 개선 못 한다"**는 같은 원칙이다. FE는 사용자가 뭘 하는지, BE는 서버가 어떻게 버티는지를 본다.

## 정리

Analytics 코드는 비즈니스 로직이 아니라 계측 코드다. 건물의 CCTV 같은 거다. 잘못 설치하면 사각지대가 생기고, 안 달면 사고 나도 뭐가 잘못됐는지 모른다. 잘못 넣으면 티가 안 나고, 안 넣으면 데이터가 없다. "되는 것 같은데 실은 안 되고 있는" 상태가 가장 위험하다.

이번에 5개 버그를 잡으면서 "Analytics도 코드 리뷰 대상이다"라는 걸 깨달았다. 특히 민감 정보 필터링은 보안 이슈라 타협 없이 잡아야 한다.
