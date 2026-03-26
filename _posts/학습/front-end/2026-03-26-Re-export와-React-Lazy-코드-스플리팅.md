---
title: "[학습] Re-export와 React.lazy 코드 스플리팅"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-26 17:00:00 +0900
categories: [학습, Frontend]
tags: [학습, React, Re-export, lazy, 코드스플리팅, 성능]
difficulty: ⭐3
render_with_liquid: false
---

## "공사 중" 페이지가 16개

[앱인토스 Admin](/posts/BE-FE-API-불일치-전면-수정/)에서 App.tsx 라우팅에 연결된 16개 페이지가 스텁이었다. "준비 중입니다" 텍스트만 표시하는 ~17줄짜리 컴포넌트. 그런데 서브디렉토리에 실제 구현체가 이미 있었다.

## Re-export로 연결

스텁 파일을 1줄짜리 re-export로 교체.

```typescript
// Before: pages/operations/BannerListPage.tsx (17줄 스텁)
export default function BannerListPage() {
  return <div>배너 목록 — 준비 중입니다</div>;
}

// After: 1줄 re-export
export { default } from './banner/BannerListPage';
```

272줄 삭제, 16줄 추가. App.tsx의 라우팅 코드는 변경 없음. 기존에 `import BannerListPage from './pages/operations/BannerListPage'`로 가져오고 있었으니, re-export가 그 경로에서 실제 구현체를 내보내면 된다.

건물 현관문에 "시공 중" 표지판을 떼고, 이미 만들어진 방으로 안내하는 표지판을 붙인 거다.

### 빌드 타임 해결

re-export는 번들러가 빌드 타임에 해결한다. 런타임 비용이 없다. `export { default } from './banner/BannerListPage'`는 번들러가 직접 원본 모듈을 연결한다.

## React.lazy와 코드 스플리팅

re-export와 함께 쓸 수 있는 패턴.

```typescript
// App.tsx
const BannerListPage = React.lazy(() => import('./pages/operations/BannerListPage'));
```

`React.lazy`는 dynamic import로 컴포넌트를 지연 로드한다. 해당 라우트에 진입할 때만 모듈을 다운로드. 앱 초기 로드 시 16개 페이지를 전부 다운로드할 필요 없다.

| 방식 | 초기 로드 | 페이지 진입 시 |
|------|----------|--------------|
| static import | 전체 번들 포함 | 즉시 |
| React.lazy | 번들 미포함 | 다운로드 후 표시 |

어드민은 내부 사용자만 쓰고, 동시 접속이 적으니 lazy loading의 이점이 크지 않다. 하지만 페이지가 16개면 전체 번들 크기 감소 효과는 있다.

## useParams로 생성/수정 분기

Admin에서 BannerFormPage가 배너 생성과 수정에 모두 쓰인다.

```typescript
const { id } = useParams<{ id: string }>();
const isEditMode = !!id;
```

URL이 `/banners/new`면 id가 없어서 생성 모드, `/banners/123`이면 수정 모드. 같은 폼 컴포넌트를 재사용한다. 서류 양식이 "신규"와 "변경" 두 가지 용도로 쓰이는 것과 같다.

## 프로젝트 적용

16개 페이지에 re-export 적용:

| 카테고리 | 페이지 |
|---------|--------|
| operations | Banner(List/Form), Notice(Form), Report(List/Detail), Maintenance |
| app-config | FeatureFlag, AppVersion, Terms |
| apps/recipe | Featured |
| audit | AuditLog |
| system | SystemStatus, AdminUser, DataExport |
