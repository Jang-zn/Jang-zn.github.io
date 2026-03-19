---
title: "[TasteNote] camelCase 마이그레이션"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-19
categories: [프로젝트, TasteNote]
tags: [프로젝트, TasteNote, React, TypeScript, API]
render_with_liquid: false
---

## 서버가 바뀌었으니 프론트도 바꾼다

[BE에서 Jackson 네이밍 전략을 바꿨다](/posts/Jackson-전략-변경과-mTLS/). Claude가 초기 세팅 때 넣어둔 snake_case 설정을 발견하고 camelCase로 통일한 것. 서버도 FE도 내가 다 만드는데, 서버가 snake_case를 내려보내면 FE에서 매번 변환해야 한다. TasteNote FE에서도 Raw 타입과 mapper에서 snake_case 키를 일일이 camelCase로 매핑하고 있었다. 이제 서버가 camelCase로 주니까 이 매핑이 전부 불필요하다.

## 4단계로 나눠서 진행

파일 24개, 변경량 366줄. 한번에 바꾸면 컴파일이 깨지니까 의존성 순서대로 4단계로 나눴다.

### 1단계: Raw 타입과 mapper

기반 레이어부터. `RawRecipe`, `RawParseJob`, `RawTerm` 같은 Raw 타입 인터페이스의 키를 snake_case에서 camelCase로 변경. mapper에서 `raw.origin_url` → `raw.originUrl` 같은 접근자도 전부 수정. 이 커밋만으로는 컴파일 에러가 난다. 의존하는 코드가 아직 구 키를 참조하고 있으니까.

### 2단계: 요청 body

API 요청 본문의 키를 변경. `origin_url` → `originUrl`, `parent_id` → `parentId`, `change_log` → `changeLog`. 레시피 편집, 버전 생성, 파싱 잡 제출, 신고 기능까지 7개 파일.

### 3단계: MSW mock 데이터

MSW mock 데이터와 핸들러의 키를 전부 변경. mock 데이터 5개 파일, 핸들러 6개 파일. 가장 변경량이 많았다. `recipes.ts` 하나만 212줄 변경.

### 4단계: 인라인 변환 코드 정리

feature 코드에 남아있던 인라인 snake_case → camelCase 변환 로직 제거. `tossLogin.ts`에서 LoginResponse 키 매핑, `useImageUpload.ts`에서 `cdn_url` 접근, `useParseJobStatus.ts`에서 Raw 접근자 — 이런 것들이 여기저기 흩어져 있었다. 서버가 camelCase를 주니까 전부 불필요.

## [Admin FE](/posts/camelCase-전환과-변환-제거/)와의 차이

Admin FE는 Axios 인터셉터에서 일괄 변환을 했기 때문에 인터셉터만 수정하면 끝이었다. TasteNote는 Raw 타입 + mapper 패턴을 쓰고 있어서 타입 정의부터 mock 데이터까지 전면 수정이 필요했다. 접근 방식이 다르니 마이그레이션 비용도 다르다.

인터셉터 일괄 변환은 마이그레이션이 쉽지만 런타임 오버헤드가 있고, Raw + mapper는 마이그레이션이 번거롭지만 타입 안전성이 높다. 결과적으로 둘 다 같은 상태가 됐다. 서버가 camelCase를 주니까 변환 자체가 필요 없는 것.

## Claude Code 협업

"서버가 camelCase로 바뀌었다, Raw 타입이랑 mapper 전부 수정해줘"라고 지시. Claude가 의존성 순서를 알아서 파악하고 4단계로 나눠서 커밋했다. 타입부터 바꾸고 → 요청 body → mock → 인라인 정리. 컴파일 에러가 커밋 단위로 연쇄적으로 해소되는 구조.

24개 파일을 30분 안에 처리. 이런 기계적인 일괄 수정은 Claude가 압도적으로 빠르다. 다만 mock 데이터는 양이 많아서 빠뜨린 부분이 없는지 MSW 돌려보고 확인했다.

## 오늘의 결과

| 항목 | 상태 |
|------|------|
| Raw 타입/mapper snake_case → camelCase | 완료 |
| API 요청 body 키 변경 | 완료 |
| MSW mock 데이터/핸들러 키 변경 | 완료 |
| 인라인 변환 코드 제거 | 완료 |
| PR #10 머지 | 완료 |

[어제](/posts/약관과-Firebase-수정/) "MSW → 실제 API 전환이 가장 큰 남은 작업"이라고 했는데, API 포맷이 통일되면서 전환이 한결 수월해졌다. Raw 타입이 서버 응답과 1:1로 대응하니까 mapper에서 키 변환 없이 그대로 매핑하면 된다.
