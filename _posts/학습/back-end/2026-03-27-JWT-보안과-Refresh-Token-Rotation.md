---
title: "[학습] JWT 보안 설계와 Refresh Token Rotation"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-27 11:00:00 +0900
categories: [학습, Backend]
tags: [학습, JWT, RefreshToken, 보안, HS256, 토큰탈취]
difficulty: ⭐5
render_with_liquid: false
---

## JWT = 서명이지 암호화가 아니다

JWT를 처음 접하면 "토큰이니까 암호화된 거겠지"라고 생각하기 쉽다. 아니다. JWT는 Base64로 인코딩되어 있을 뿐이다. 누구나 디코딩해서 안의 내용을 읽을 수 있다.

여권을 생각하면 된다. 여권에 적힌 이름, 국적, 생년월일은 누구나 펼쳐서 읽을 수 있다. 하지만 위조 방지 도장이 찍혀 있어서 내용을 고치면 바로 발각된다. JWT도 마찬가지다. payload는 공개 정보고, signature가 위조 방지 도장이다.

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIn0.SflKxwRJSMe...
│── Header ──────────│── Payload ──────────────│── Signature ──│
```

Header와 Payload는 Base64 디코딩만 하면 그대로 읽힌다. 그래서 비밀번호, 주민번호 같은 민감정보를 payload에 넣으면 안 된다. JWT는 **무결성**(변조 감지)을 보장하지, **기밀성**(내용 숨김)을 보장하지 않는다.

### alg:none 공격

JWT Header의 `alg` 필드를 `none`으로 바꿔서 서명 검증 자체를 우회하는 고전적 공격이 있다. 방어는 간단하다. 토큰 생성/파싱 시 알고리즘을 코드에서 명시적으로 고정한다.

```java
Jwts.builder()
    .signWith(secretKey, Jwts.SIG.HS256)  // 알고리즘 고정
    .compact();
```

라이브러리가 알아서 해주겠지 하면 안 된다.

---

## HS256 vs RS256

JWT 서명 알고리즘은 대칭키(HS256)와 비대칭키(RS256) 두 갈래다.

| 구분 | HS256 (대칭키) | RS256 (비대칭키) |
|------|---------------|-----------------|
| 키 구성 | Secret Key 1개 | Private Key + Public Key |
| 서명 | Secret Key로 서명 | Private Key로 서명 |
| 검증 | 같은 Secret Key로 검증 | Public Key로 검증 |
| 적합한 환경 | 단일 서버, 모놀리식 | MSA, 외부 서비스 검증 필요 |
| 성능 | 빠름 | 상대적으로 느림 |

집 열쇠가 하나인 원룸(HS256)이면 그 열쇠 하나로 잠그고 연다. 건물 관리 사무실(RS256)이면 마스터키(Private)로 잠그고, 각 세대에 배포한 카드키(Public)로 연다. 서버가 한 대면 HS256이면 충분하다.

---

## 시크릿 키 관리

JWT 보안에서 가장 흔한 실수가 시크릿 키 관리다.

### 하드코딩하면 Git에 올라간다

```yaml
# 이러면 안 된다
jwt:
  secret: ${JWT_SECRET:default-secret-key-for-development}
```

기본값이 있으면 환경변수 설정을 까먹어도 서버가 뜬다. 그 기본값이 프로덕션에 배포되면 끝이다. 기본값을 제거해서 환경변수가 없으면 서버 시작 자체를 실패하게 만든다.

```yaml
jwt:
  secret: ${JWT_SECRET}  # 기본값 없음 → 미설정 시 즉시 에러
```

### 32바이트 최소 기준

HMAC-SHA256은 최소 256비트(32바이트) 키를 요구한다. 짧은 키는 브루트포스에 취약하다. 서버 시작 시 길이를 검증한다.

```java
@PostConstruct
void validateSecretKey() {
    if (secret.getBytes(UTF_8).length < 32) {
        throw new IllegalArgumentException("JWT secret must be at least 32 bytes");
    }
}
```

환경변수 미설정이면 서버가 안 뜨고, 키가 짧아도 서버가 안 뜬다. 둘 다 프로덕션에서 터지는 것보다 낫다.

| 위험 | 방어 |
|------|------|
| 코드에 하드코딩 | 환경변수로 주입, 기본값 제거 |
| 짧은 키 (< 32바이트) | @PostConstruct 길이 검증 |
| 모든 환경에서 같은 키 | 환경별 키 분리 |
| 키 로테이션 없음 | 주기적 키 교체 |

---

## Access Token + Refresh Token 구조

회사에 출근하면 두 가지를 받는다. 출입증(Access Token)은 매일 아침 로비에서 발급한다. 잃어버려도 하루면 만료된다. 사원증(Refresh Token)은 입사 시 1회 발급한다. 출입증이 만료되면 사원증을 보여주고 새 출입증을 받는다.

| 구분 | Access Token | Refresh Token |
|------|-------------|---------------|
| 수명 | 짧음 (1시간) | 김 (7~30일) |
| 용도 | API 호출 인증 | Access Token 재발급 |
| 저장 위치 | 메모리 / Authorization 헤더 | HttpOnly 쿠키 / 서버 DB |
| 탈취 시 영향 | 만료까지만 피해 | 전체 세션 탈취 가능 |

왜 분리하는가. AT를 도둑맞아도 수명이 1시간이라 피해가 제한된다. RT는 수명이 길지만 서버에 안전하게 보관하고, API 호출에 직접 사용하지 않는다. 피해 범위를 토큰 종류별로 분리하는 거다.

---

## Refresh Token Rotation

RT가 고정값이면 한번 탈취되면 만료일까지 무한으로 AT를 발급받을 수 있다. 서버는 정상 요청과 공격 요청을 구분할 방법이 없다.

Rotation은 간단하다. **refresh할 때마다 새 RT를 발급하고, 이전 RT를 즉시 폐기한다.** Redis에는 항상 현재 유효한 RT 1개만 저장된다.

### 탈취 감지 로직

핵심은 "이미 사용된 RT가 다시 오면 탈취"로 간주하는 거다.

1. 사용자가 RT1을 가지고 있다
2. 공격자가 RT1을 탈취한다
3. 공격자가 RT1으로 refresh → 서버는 RT2를 발급하고 RT1을 폐기한다
4. 정상 사용자가 RT1으로 refresh → 서버에 저장된 건 RT2인데 RT1이 왔다
5. **불일치 감지 → 해당 사용자의 모든 토큰 무효화**

```java
Optional<String> storedToken = cachePort.get("refresh:" + userId, String.class);

if (storedToken.isPresent()
        && !storedToken.get().equals(request.refreshToken())) {
    // 이미 사용된 토큰이 다시 왔다 = 탈취
    cachePort.evict("refresh:" + userId);  // 전체 무효화
    throw new BusinessException(ErrorCode.INVALID_REFRESH_TOKEN);
}

// 새 토큰 발급 (Rotation)
String newRefreshToken = jwtTokenProvider.createRefreshToken(userId);
cachePort.put("refresh:" + userId, newRefreshToken, Duration.ofMillis(rtExpiration));
```

정상 사용자가 재로그인해야 하는 건 불편하지만, 공격자의 지속적 접근을 차단하는 게 더 중요하다. 재로그인은 한 번이면 끝이고, 탈취 피해는 누적된다.

| 항목 | Rotation 없음 | Rotation 있음 |
|------|-------------|-------------|
| 탈취된 RT 수명 | 만료일까지 유효 | 1회 사용 후 무효 |
| 탈취 감지 | 불가능 | RT 불일치로 즉시 감지 |
| 탈취 대응 | 만료 대기뿐 | Redis 삭제로 전체 세션 무효화 |

### FE 변경 필수

Rotation을 도입하면 FE도 바뀌어야 한다. refresh 응답에 새 RT가 포함되고, FE는 이걸 반드시 저장해야 한다. 안 하면 다음 refresh에서 이전 RT를 보내게 되고, 서버는 탈취로 오인해서 세션을 무효화한다.

---

## 로그아웃 = RT 폐기

Rotation 체계에서 로그아웃은 간단하다. Redis에서 RT를 삭제하면 된다.

```java
@PostMapping("/logout")
public ResponseEntity<ApiResponse<Void>> logout(
        @AuthenticationPrincipal UserPrincipal principal) {
    cachePort.evict("refresh:" + principal.getUserId());
    return ResponseEntity.ok(ApiResponse.ok());
}
```

RT가 Redis에서 사라지면 이후 refresh 요청은 전부 실패한다. 재로그인 필수. 여기서 주의할 점이 있다. `/auth/**`를 일괄 `permitAll`로 열면 logout도 인증 없이 호출 가능해진다. logout은 `authenticated`로 JWT 필수로 걸어야 한다.

```java
// login, refresh는 permitAll
// logout은 authenticated → JWT 필수
.requestMatchers("/api/v1/auth/toss-login", "/api/v1/auth/refresh").permitAll()
```

---

## 앱인토스에 적용

[보안 강화 프로젝트 포스트](/posts/보안-강화-IDOR부터-Webhook까지/)에서 JWT + Refresh Token Rotation을 적용했다. AT 1시간, RT 7일. Redis에 현재 유효 RT만 저장하고, 불일치 시 전체 세션 무효화. Admin API는 시크릿 키를 별도로 분리해서 한쪽 키가 유출돼도 다른 쪽은 안전하게 만들었다.
