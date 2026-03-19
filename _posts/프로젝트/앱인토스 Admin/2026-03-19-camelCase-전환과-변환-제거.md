---
title: "[앱인토스 Admin] camelCase 전환과 변환 로직 제거"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-19
categories: [프로젝트, 앱인토스 Admin]
tags: [프로젝트, 앱인토스 Admin, React, API, TypeScript]
render_with_liquid: false
---

## 어제 만든 걸 오늘 지웠다

[어제](/posts/API-클라이언트-인프라-구축/) Axios 인터셉터에 snake_case ↔ camelCase 자동 변환을 넣었다. `caseConverter` 유틸이 재귀적으로 중첩 객체까지 전부 변환하는 구조. 오늘 지웠다.

[BE에서 Jackson 전략을 바꿔서](/posts/Jackson-전략-변경과-mTLS/) 서버가 camelCase로 내려주게 됐기 때문이다. 원래 BE의 snake_case 설정 자체가 Claude가 초기 세팅 때 넣어둔 거였다. 서버도 FE도 내가 다 개발하는데 굳이 snake_case를 쓸 이유가 없다. 서버를 camelCase로 통일하면 클라이언트에서 변환할 이유가 없어진다.

## 제거한 것

- `caseConverter.ts` 파일 전체 삭제
- Request 인터셉터에서 `toSnakeCase(data/params)` 호출 제거
- Response 인터셉터에서 `toCamelCase()` 호출 제거
- 에러 핸들러에서 `toCamelCase(error)` → 직접 타입 캐스트
- 토큰 리프레시 요청 body: `{ refresh_token }` → `{ refreshToken }`

인터셉터가 한결 가벼워졌다. 재귀 객체 순회가 매 요청/응답마다 돌고 있었는데, 페이지네이션 응답처럼 데이터가 클 때 오버헤드가 있었을 거다.

## 위험 요소

서버 쪽 마이그레이션이 완료되지 않은 엔드포인트가 있으면 런타임에서 터진다. snake_case 키가 들어오면 타입 불일치로 `undefined`가 된다. BE PR이 먼저 머지된 걸 확인하고 FE PR을 올렸다.

## Claude Code 협업

"서버가 camelCase로 바뀌었으니 변환 로직 제거해줘"라고 한 줄 지시. Claude가 `caseConverter.ts` 삭제부터 인터셉터 수정, refresh token body 키 변경까지 한번에 처리. 단순 제거 작업이라 Claude가 놓치는 부분 없이 깔끔하게 했다.

## 오늘의 결과

| 항목 | 상태 |
|------|------|
| caseConverter 유틸 삭제 | 완료 |
| 인터셉터 변환 로직 제거 | 완료 |
| refresh token body 키 변경 | 완료 |
| PR #2 머지 | 완료 |

어제 "API 연동 인프라 완성"이었는데, 오늘 그 인프라에서 불필요한 레이어를 하나 벗겨낸 거다. 서버가 클라이언트 친화적인 포맷을 내려주면 클라이언트 코드가 단순해진다. 당연한 얘기지만, 실제로 체감하니 좀 다르다.
