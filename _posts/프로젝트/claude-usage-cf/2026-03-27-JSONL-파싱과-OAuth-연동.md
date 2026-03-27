---
title: "[claude-usage-cf] 569MB JSONL 파싱과 OAuth API 연동"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-27 21:30:00 +0900
categories: [프로젝트, claude-usage-cf]
tags: [프로젝트, claude-usage-cf, JSONL, OAuth, 증분파싱, 집계, Python]
render_with_liquid: false
---

> 이전 글: [Claude Code 토큰 대시보드 — 소개](/posts/토큰-대시보드-소개/)

## JSONL 구조

Claude Code는 `~/.claude/projects/` 아래에 프로젝트별 디렉토리를 만든다. 각 디렉토리 안에 JSONL 파일이 있다. 한 줄이 하나의 JSON 객체.

줄마다 `type` 필드가 있다. `"human"`, `"assistant"`, `"system"` 등. 토큰 정보는 `"assistant"` 타입 메시지에만 들어있다. `message.usage` 안에 `input_tokens`, `output_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens`가 있다.

모델명은 `message.model`에 있다. `"claude-opus-4-6"` 같은 원시 ID로 들어온다. 이걸 `"opus-4.6"`으로 정규화해서 쓴다.

열심히 쓰면 파일이 커진다. 내 경우 569MB. 매일 Claude Code를 쓰면 JSONL이 계속 쌓인다. 한 줄에 수 KB씩, 하루에 수천 줄. 몇 달 쓰면 수백 MB는 금방이다.

## 증분 파싱

569MB를 매 초 처음부터 읽으면 안 된다. 한 번에 몇 초 걸린다. 1초마다 갱신하는 대시보드에서 그건 불가능하다.

byte offset을 기록한다. 파일을 읽은 후 `f.tell()`로 현재 위치를 저장. 다음에는 `f.seek(offset)`으로 그 위치부터 이어 읽는다.

책갈피다. 500페이지짜리 책을 매번 1페이지부터 읽는 사람은 없다. 지난번에 읽은 데까지 페이지를 끼워두고, 다음에는 거기부터 읽는 거다.

```python
f.seek(start_offset)
while True:
    line = f.readline()
    if not line:
        break
    # 파싱
_file_offsets[filepath] = f.tell()
```

증분 파싱 덕에 한 번의 리프레시가 5-10ms면 끝난다. 새로 추가된 줄만 읽으니까.

### fast pre-filter

JSON 파싱 자체도 비용이다. 모든 줄을 `json.loads`에 넣으면 느리다. 그래서 JSON 파싱 전에 문자열 필터를 먼저 돌린다.

```python
_ASSISTANT_MARKERS = ('"type":"assistant"', '"type": "assistant"')

if not any(m in line for m in _ASSISTANT_MARKERS):
    continue
```

`"assistant"` 문자열이 포함되지 않은 줄은 JSON 파싱 없이 건너뛴다. human 메시지, system 메시지가 전체의 절반 이상인데, 이걸 문자열 검사 한 줄로 걸러낸다. 공백 유무까지 두 패턴으로 잡는다. JSON 직렬화가 어떤 포맷으로 나올지 모르니까.

### 프로젝트명 추출

프로젝트 이름은 파일 경로에서 뽑아낸다. JSONL 파일 안에 프로젝트명이 없다. 디렉토리명이 곧 프로젝트 ID다.

경로가 `~/.claude/projects/-Users-jang-projects-app-be/conversation.jsonl` 같은 형태다. `-Users-jang-projects-`에서 `projects` 뒤를 자르면 `app-be`가 된다.

정규식 두 개로 처리한다. `-projects-(.+)`와 `-workspace-(.+)`. 이 두 패턴이 안 맞으면 홈 디렉토리 구조를 가정하고 fallback으로 파싱.

## 턴(Turn) 카운팅

Token Usage 패널에 "평균 토큰/턴"을 보여준다. "이 모델로 한 번 질문하면 평균 얼마나 먹는가." 이게 남은 쿼터를 가늠하는 핵심 지표다.

근데 "질문 1번"을 어떻게 정의하느냐가 문제다.

처음에는 API 호출 횟수 기준으로 했다. JSONL에 assistant 메시지가 하나면 1턴. 단순하고 직관적이다.

문제는 tool_use 체인이다. Claude Code에 "이 파일 읽고 수정해"라고 하면, 내부적으로 파일 읽기 → 분석 → 편집 → 확인 같은 과정을 거친다. assistant 메시지가 5-10개 연속으로 나온다. 사용자 입장에서는 질문 1개인데, API 호출은 5-10회.

이 상태로 평균을 내면 토큰/턴이 비정상적으로 낮게 나온다. "Opus 턴당 2만 토큰"이라고 뜨는데, 실제 체감은 한 번 물어보면 15만은 쓰는 느낌. 수치가 안 맞았다.

해결: **30초 gap threshold**. 같은 세션 내에서 이전 assistant 메시지와 30초 이상 간격이 나면 새 턴으로 친다. tool_use 체인은 보통 수 초 간격으로 연속 실행되니까, 30초면 충분히 구분된다. 사용자가 다음 질문을 타이핑하는 데 최소 30초는 걸린다는 가정이다.

```python
_TURN_GAP = 30  # seconds
last_ts = _session_last.get(sid)
if last_ts is None or (rec.timestamp - last_ts).total_seconds() > _TURN_GAP:
    models[rec.model].turn_count += 1
```

이걸 적용하니까 수치가 체감과 맞았다. Opus 턴당 10-15만, Sonnet 턴당 5-8만 정도. 처음에 API 호출 수 기준으로 계산했다가 수치가 이상해서 발견한 버그다.

## Aggregator

Aggregator는 모든 데이터 소스를 합쳐서 `AggregatedUsage` 하나로 만든다.

핵심 기능:

1. **기간 필터링**: day/week/month/session에 따라 레코드를 거른다
2. **모델별 집계**: 각 모델의 총 토큰, 요청 수, 턴 수
3. **프로젝트별 집계**: 프로젝트 이름별 토큰 합산
4. **일별 집계**: 날짜별 토큰 합산 (Daily Usage 차트용)
5. **5시간 rolling window**: OAuth 퍼센트 계산용

session 모드(`5` 키)는 좀 다르다. 날짜 기반이 아니라 "지금부터 5시간 전" 기준 rolling window다. 현재 시각에서 5시간을 빼서 그 이후 레코드만 가져온다.

day/week/month는 로컬 타임존 기준 달력 날짜로 필터한다. "오늘"은 `datetime.now().date()` 기준. UTC가 아니라 사용자 로컬. 이거 처음에 UTC로 했다가 날짜가 하루 밀리는 버그가 있었다. 한국 시간 새벽 3시에 쓰면 UTC 기준으로는 전날인 거다.

### _record_cache

증분 파싱한 레코드를 `_record_cache`에 쌓는다. 매 리프레시마다 새로 파싱하는 게 아니라, 이전에 파싱한 결과를 재사용한다. 새 레코드만 추가.

이 캐시가 무한히 커지면 문제가 된다. 장시간 돌리면 메모리가 계속 늘어난다. lookback 기간(30일) 초과 레코드를 trim하는 로직이 있다. 이건 [최적화 포스트](/posts/Windows-버그와-최적화/)에서 자세히 다룬다.

## OAuth API 연동

쿼터 퍼센트는 로컬 계산만으로 안 된다. Anthropic이 내부적으로 쓰는 한도값을 모르니까.

"오늘 10만 토큰 썼는데, 한도가 100만인지 200만인지 모르겠다." 분자는 아는데 분모를 모르는 상태. 퍼센트를 계산할 수 없다.

### 엔드포인트

`/api/oauth/usage` 엔드포인트를 호출한다. beta 헤더(`anthropic-beta: oauth-2025-04-20`)가 필요하다. 응답에 3개 쿼터 윈도우가 들어온다.

| 윈도우 | 설명 |
|--------|------|
| `five_hour` | 5시간 세션 쿼터 |
| `seven_day` | 7일 전체 모델 쿼터 |
| `seven_day_sonnet` | 7일 Sonnet 전용 쿼터 |

각 윈도우에 `utilization`(사용률 %)과 `resets_at`(리셋 시각 ISO 문자열)이 들어온다.

### 한도 역산

API가 "지금 42% 썼다"고 알려주면, 로컬에서 계산한 토큰 수치로 한도를 역산한다.

```
한도 = 사용량 / (사용률 / 100)
```

5시간 윈도우에서 420만 토큰을 썼고 사용률이 42%면, 한도는 1000만이다. 이 한도값을 저장해둔다.

이후에는 로컬 토큰 수치를 한도로 나눠서 퍼센트를 계산한다. API를 매 초 호출할 필요가 없다.

수도 계량기를 확인하겠다고 1초마다 수도국에 전화하는 짓은 안 한다. 처음 한 번 한도를 알아내고, 이후에는 내가 쓴 양을 직접 더하면 된다.

### 사용률 0% 처리

리셋 직후에는 사용률이 0%다. `한도 = 사용량 / 0`은 division by zero. 최소 1%로 가정해서 역산한다. `max(utilization, 1.0)`.

## 재동기화 전략

한도는 한 번 역산하면 끝이 아니다. 시간이 지나면 실제 값과 벌어질 수 있다.

### 30분 주기 재호출

`_SYNC_INTERVAL = 1800`. 마지막 호출로부터 30분이 지나면 API를 다시 호출한다. 로컬 계산이 실제와 얼마나 차이나는지 보정하는 주기.

### 리셋 감지

쿼터 윈도우가 리셋되면 한도가 바뀔 수 있다. `resets_at` 시각이 현재보다 과거가 되면 리셋이 발생한 거다. 이 경우 저장된 한도를 버리고 API를 즉시 재호출한다.

```python
for info in (_raw.five_hour, _raw.seven_day, _raw.seven_day_sonnet):
    ts = _resets_at_ts(info)
    if ts and _last_fetch < ts <= now:
        return True  # should refetch
```

윈도우의 `resets_at`이 바뀌면 한도도 `None`으로 초기화한다. 새 윈도우에서 새로 역산한다.

### stale-on-error

API가 실패하면? 마지막에 성공한 값을 그대로 유지한다. 네트워크가 잠깐 끊겨도 대시보드가 먹통이 되면 안 된다. 로컬 계산이라도 계속 돌아간다.

### 스레드 안전성

`threading.Lock`으로 보호한다. `refresh_data`가 백그라운드 스레드에서 돌기 때문에, OAuth 상태를 동시에 읽고 쓰면 안 된다.

```python
_lock = threading.Lock()

def get_oauth_usage(...):
    with _lock:
        now = time.time()
        if force or _should_refetch(now):
            _do_fetch()
        # ... 계산
```

모듈 레벨 전역 변수(`_raw`, `_last_fetch`, `_limit_*`)를 Lock 안에서만 접근한다.

## 리셋 직후 0% 버그

5시간 윈도우가 리셋된 직후, 아직 토큰을 안 썼으면 사용률이 0%다.

문제: 게이지 바가 0%로 뜬다. 사용자 입장에서 "아까까지 80%였는데 갑자기 0%?" 혼란스럽다. 쿼터가 리셋된 건지, 데이터가 날아간 건지 구분이 안 된다.

최소 1%로 표시하기로 했다. 게이지 바에 1%라도 색이 들어가면 "쿼터가 존재하고, 거의 안 썼다"는 신호가 된다. 0%는 "데이터 없음"과 시각적으로 구분이 안 되니까.

```python
display_pct = max(pct, 1.0)  # 0% 리셋 직후에도 최소 1%로 표시
```

작은 디테일이지만, 대시보드는 "한눈에 파악"이 목적이다. 혼란을 주는 건 없어야 한다.

## 인증 토큰 획득

OAuth API를 호출하려면 인증 토큰이 필요하다. Claude Code 로그인 정보를 가져다 쓴다.

### macOS

`security find-generic-password` CLI로 Keychain에서 읽는다.

```python
raw = subprocess.check_output(
    ["security", "find-generic-password", "-s", "Claude Code-credentials", "-w"],
    stderr=subprocess.DEVNULL, timeout=3,
).decode().strip()
token = json.loads(raw)["claudeAiOauth"]["accessToken"]
```

Keychain에 `"Claude Code-credentials"` 서비스명으로 저장된 JSON 문자열에서 `accessToken`을 꺼낸다.

### Windows / Linux

`~/.claude/.credentials.json` 파일에서 읽는다. `CLAUDE_CONFIG_DIR` 환경변수가 설정되어 있으면 그 경로를 쓴다.

런타임에만 읽고, 별도 파일에 저장하지 않는다. 토큰이 코드나 설정 파일에 박히면 안 되니까. macOS Keychain이 안 되면 JSON 파일 fallback으로 간다.

---

다음 포스트에서 Windows `os.kill` 버그, 1초 리프레시 최적화, `/simplify`가 잡아낸 것들을 다룬다.

> 다음 글: [Windows 프로세스 킬 버그와 1초 리프레시 최적화](/posts/Windows-버그와-최적화/)
