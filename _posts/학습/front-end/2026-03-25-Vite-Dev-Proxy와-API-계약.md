---
title: "[학습] Vite Dev Proxy와 FE-BE API 계약"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-25 23:00:00 +0900
categories: [학습, Frontend]
tags: [학습, Vite, Proxy, CORS, API, 계약]
render_with_liquid: false
---

## 로컬에서 API를 쏘면 CORS가 터진다

[앱인토스 Admin](/posts/BE-FE-API-불일치-전면-수정/)에서 로컬 개발 중 BE API를 호출하면 브라우저가 CORS 에러를 뱉는다. FE는 `localhost:5173`, BE는 `localhost:8083`. 포트가 다르면 다른 출처(origin)다.

## Vite Dev Proxy

Vite 개발 서버가 프록시 역할을 한다. FE → Vite 서버(같은 출처) → BE. 브라우저는 같은 출처로 요청한다고 인식하니까 CORS 문제가 없다.

```typescript
// vite.config.ts
server: {
  proxy: {
    '/api': 'http://localhost:8083',
    '/admin': 'http://localhost:8083',
  }
}
```

`/api`로 시작하는 요청은 8083 포트로 전달. 편의점 택배 접수처와 같다. 고객은 편의점(Vite)에 맡기고, 편의점이 택배사(BE)에 전달한다.

### 포트 변경 사고

BE 포트를 8081에서 8083으로 바꿨는데 Vite proxy를 안 바꿔서 API가 안 됐다. proxy 대상 주소가 잘못되면 502가 나온다. 편의점이 폐업한 택배사로 보내는 꼴.

## 죽은 코드 제거

Admin FE에 BE에 없는 API 엔드포인트가 6개 있었다.

- `DASHBOARD.TRENDS` — 대시보드 트렌드 차트
- audit export — 감사 로그 내보내기
- parse job recover — 파싱 작업 복구
- ingredient bulk import — 재료 일괄 가져오기
- 기타 2개

BE 컨트롤러에 없는 엔드포인트를 FE가 호출하고 있었다. API 상수 → API 함수 → hooks → pages까지 4개 레이어를 타고 들어가 있었다. 위에서부터 순서대로 삭제.

## API 계약의 중요성

BE-FE를 동시에 개발할 때 "API 계약"이 없으면 이런 불일치가 쌓인다.

| 방법 | 장점 | 단점 |
|------|------|------|
| Swagger/OpenAPI | 자동 생성, 항상 최신 | 설정 필요 |
| 공유 문서 | 논의하며 작성 가능 | 코드와 싱크 안 맞을 수 있음 |
| BE 먼저 구현 | FE가 실제 응답 기반 | BE 대기 시간 |
| MSW 선행 | FE가 먼저 개발 가능 | 스펙 합의 필수 |

이번 프로젝트에서는 Swagger를 띄워서 FE가 실제 응답을 보고 타입을 정의하는 방식으로 바꿨다. "아마 이렇겠지"로 타입을 만들면 [Spring Page 불일치](/posts/Spring-Page-응답과-FE-타입-매칭/) 같은 문제가 생긴다.

## BE 변경의 FE 전파 경로

BE에서 엔드포인트 하나를 제거하면 FE에서 4곳을 수정해야 한다.

```
API 상수 (endpoints.ts) → API 함수 (*.api.ts) → Hook (use*.ts) → Page (*.tsx)
```

이 체인이 길수록 죽은 코드가 남기 쉽다. 상수를 지웠는데 hook에서 직접 URL을 쓰고 있으면 안 잡힌다. TypeScript의 타입 체크가 컴파일 타임에 잡아주긴 하지만, `any` 타입이 끼어있으면 통과해버린다.
