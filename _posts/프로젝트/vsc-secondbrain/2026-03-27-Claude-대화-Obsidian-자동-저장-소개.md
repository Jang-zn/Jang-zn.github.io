---
title: "[vsc-secondbrain] Claude Code 대화를 Obsidian에 자동 저장 — 소개"
author:
  name: 코드대장간
  link: https://github.com/Jang-zn
date: 2026-03-27 20:00:00 +0900
categories: [프로젝트, vsc-secondbrain]
tags: [프로젝트, vsc-secondbrain, VSCode, Obsidian, ClaudeCode, TypeScript, Gemini]
render_with_liquid: false
---

## 왜 만들었나

Claude Code로 하루에 수십 개 대화를 한다. 뭘 했는지 3시간만 지나면 까먹는다.

대화 내용은 `~/.claude/projects/` 아래에 JSONL 파일로 쌓인다. 한 줄에 JSON 한 덩어리씩. thinking 블록, tool_use, file-history-snapshot 같은 게 줄줄이 들어있다. 사람이 읽을 수 있는 물건이 아니다.

냉동실에 라벨 없이 쌓아둔 반찬통 같은 거다. 열어봐야 뭔지 안다. 6개월 전 반찬인지 어제 반찬인지 구분도 안 된다. "아 그때 그거 어떻게 해결했더라" 하고 찾으려면 JSONL 파일을 하나씩 열어서 JSON을 눈으로 파싱해야 한다. 말이 안 된다.

Obsidian에 마크다운으로 자동 정리되면 좋겠다고 생각했다. 대화 끝나면 요약해서 저장하고, 기존 문서랑 위키링크로 연결까지. 그래서 VS Code/Cursor 확장을 만들었다. 이름은 `vsc-secondbrain`.

## 전체 아키텍처

7단계 파이프라인이다.

1. **chokidar**가 `~/.claude/projects/**/*.jsonl` 감시
2. 파일 변경 감지되면 **dirty 목록**에 추가
3. **스케줄러**가 20분 슬롯(:00/:20/:40)에 dirty 파일들 처리 시작
4. **JSONL 파서**가 대화 메시지 추출 (노이즈 필터링)
5. **Gemini Flash Lite**가 요약 생성 (주제별 분리, JSON 구조화)
6. **vault 인덱싱** → keyTopics와 기존 노트 매칭 → `[[wikilinks]]` 생성
7. Obsidian vault에 **마크다운 저장**

비서가 회의록을 정리해서 파일캐비닛에 넣어주는 거다. 회의 끝나면 알아서 핵심만 뽑아서 정리하고, 관련 문서에 포스트잇으로 참조 표시까지 붙여준다. 나는 회의만 하면 된다.

의존성은 세 개다. `chokidar`(파일 감시), `@google/generative-ai`(Gemini API), `p-queue`(동시성 제어). VS Code 확장이니까 `vscode` API는 기본 포함. 가벼운 구성이다.

## JSONL 파싱의 현실

`~/.claude/projects/` 아래 구조부터 보자. 프로젝트 경로를 `-`로 인코딩한 폴더명 안에 세션별 JSONL 파일이 생긴다. 한 파일이 하나의 대화 세션이다.

JSONL 레코드 타입은 다섯 가지다.

| 타입 | 내용 | 처리 |
|------|------|------|
| `user` | 사용자 메시지 | 텍스트 추출 |
| `assistant` | Claude 응답 | 텍스트 + tool_use 이름 추출 |
| `system` | 시스템 메시지 | 스킵 |
| `file-history-snapshot` | 파일 변경 스냅샷 | 스킵 |
| `progress` | 진행 상태 | 스킵 |

user와 assistant만 의미 있다. 나머지는 전부 노이즈다.

assistant 레코드 안에는 content 블록이 배열로 들어있다. `text`, `thinking`, `tool_use`, `tool_result` 같은 타입이 섞여 있다. thinking 블록은 Claude의 내부 추론이라 요약에 필요 없다. tool_use는 도구 이름만 뽑아서 "어떤 도구를 썼는지" 메타데이터로 기록한다. 실제 텍스트 내용은 text 블록에서만 가져온다.

```typescript
for (const block of content) {
  if (block.type === 'text') { parts.push(block.text); }
  else if (block.type === 'tool_use') { tools.push(block.name); }
  // thinking 블록은 무시
}
```

user 레코드도 필터링이 필요하다. `isMeta`가 true인 건 `/clear` 같은 내부 명령이다. content가 배열일 때 `<teammate-message`로 시작하는 텍스트 블록은 Claude Code Teams의 서브에이전트 통신이다. 둘 다 스킵.

이렇게 필터링하고 나면 사람이 읽을 수 있는 대화 텍스트가 남는다. 이게 Gemini에 들어간다.

한 가지 더. JSONL 파일이 클 수 있다. 장시간 대화는 수천 줄이 넘어간다. Gemini에 통째로 넘기면 토큰 제한에 걸린다. 그래서 대화 텍스트를 60,000자 기준으로 자른다. 잘려도 앞부분은 이전 처리에서 이미 요약됐을 가능성이 높다. 증분 처리와 맞물리면 문제가 없다.

## Gemini 요약

왜 Gemini Flash Lite인가. 세 가지 이유다.

1. **싸다.** 대화 요약은 하루에 수십 번 돌아간다. 비싼 모델 쓰면 API 비용이 눈에 띈다.
2. **빠르다.** 요약 하나에 2-3초면 끝난다. 사용자 경험에 영향을 주지 않는다.
3. **충분하다.** 대화 요약은 고급 추론이 필요한 작업이 아니다. 주제 분류하고 핵심 문장 뽑는 정도면 된다.

Gemini에 넘기는 프롬프트가 꽤 길다. 핵심은 이거다.

- 대화를 **주제별로 1~5개**로 분리해라
- 각 주제마다 **JSON 구조**로 반환해라: title, summary, keyTopics, decisions, codeChanges, tags, messageIndices
- "문서화할 가치가 없는 대화"는 빈 배열로 반환해라
- 아직 진행 중인 대화는 `incomplete: true`로 표시해라

문서화 기준이 중요하다. "사용자의 문제 제기 → 해결방안 구체화 → 실행"이 완성된 사이클만 노트로 만든다. 단순히 코드를 탐색하고 읽기만 한 건 노트 대상이 아니다. incomplete 판단도 구체적으로 지시했다. Claude가 "분석하겠습니다", "시작하겠습니다"만 말하고 실제 결과가 없으면 incomplete다.

이 기준이 없으면 "파일 3개 읽었습니다" 같은 쓸데없는 노트가 대량 생산된다. 필터링이 요약만큼 중요하다.

messageIndices는 주제별로 어떤 메시지가 해당하는지 인덱스 번호를 매핑한다. 나중에 NoteWriter가 해당 메시지만 뽑아서 "Full Conversation" 섹션에 넣는다.

## Wikilinks

Obsidian의 핵심 기능이 `[[wikilinks]]`다. 노트끼리 연결해서 지식 그래프를 만든다. 이걸 자동화하고 싶었다.

VaultIndex 클래스가 vault 폴더를 재귀적으로 순회하면서 마크다운 파일의 frontmatter에서 title을 수집한다. 이게 기존 노트 목록이다.

LinkMatcher가 Gemini 요약의 keyTopics와 기존 노트 제목을 대조한다. 대소문자 무시하고 부분 매칭. "TypeScript"라는 keyTopic이 있고 vault에 "TypeScript 기초"라는 노트가 있으면 `[[TypeScript 기초]]`로 링크를 건다.

```typescript
const link = matchedLinks.find(l =>
  l.toLowerCase().includes(topic.toLowerCase()) ||
  topic.toLowerCase().includes(l.toLowerCase())
);
return link ? `- [[${link}]] - ${topic}` : `- ${topic}`;
```

매칭이 안 되는 keyTopic은 일반 텍스트로 남는다. 나중에 해당 주제의 노트가 생기면 그때 연결될 수 있다. Obsidian이 깨진 링크를 자동 감지해주니까 수동으로 연결할 수도 있다.

이게 누적되면 대화 주제들 사이의 관계가 보인다. "FileLock" 노트를 열면 연결된 대화들이 쭉 나온다. 언제 어떤 맥락에서 이 주제를 다뤘는지 한눈에 보인다.

## 생성되는 노트 구조

마크다운 파일 구조는 이렇다.

**YAML frontmatter:**
- `title`: 요약 제목
- `date`: 대화 시작 시각
- `tags`: claude, 프로젝트명, 요약 태그
- `project`: 프로젝트명
- `session_id`: Claude Code 세션 ID
- `git_branch`: 작업 브랜치 (있으면)

**본문 섹션:**
- **Summary** — 5~10문장 요약. 3문장마다 단락 구분 자동 삽입
- **Key Topics** — 기술/개념명 목록. vault 매칭 시 wikilink 포함
- **Decisions** — 이 대화에서 내린 결정 사항
- **Code Changes** — 수정/생성된 파일과 변경 내용
- **Related Notes** — 매칭된 vault 노트 링크
- **Full Conversation** — 해당 주제의 원본 대화 (인덱스 기반 필터링)

파일 생성은 `openSync('wx')`로 원자적으로 한다. 파일이 이미 있으면 `EEXIST` 에러가 나고, 기존 파일 경로를 그대로 반환한다. 다른 인스턴스가 동일 노트를 동시에 만들려 해도 중복이 생기지 않는다. 이건 [동시성편](/posts/동시성-문제-해결기)에서 자세히 다룬다.

파일명은 `{HH-mm}-{slugified-title}.md` 형식이다. 날짜 폴더 안에 시각과 제목으로 구분된다.

## Claude Code 협업

하루 만에 34커밋. 거의 전부 Claude Code와 페어 프로그래밍이다.

**초기 탐색 단계**가 가장 효율적이었다. `~/.claude/` 디렉토리 구조를 Claude Code에 분석시키는 것부터 시작했다. JSONL 포맷, 레코드 타입, content 블록 구조를 Claude Code가 직접 파일을 열어서 파악했다. 이걸 기반으로 파서 뼈대를 만들었다.

큰 단위로 지시하는 게 맞았다. "JSONL 파서 만들어", "Gemini 요약기 만들어", "마크다운 노트 작성기 만들어". 각 모듈이 독립적이라 한 번에 하나씩 만들고 테스트할 수 있었다.

VS Code API 연동은 Claude Code가 잘하는 영역이다. 확장 활성화, 명령어 등록, 설정 스키마, 상태바 표시, API 키 시크릿 저장. 문서를 찾아보고 바로 코드를 뱉는다. 이 부분에서 시간을 많이 절약했다.

반면 설계 결정은 내가 했다. "JSONL에서 뭘 필터링할지", "요약 프롬프트에 어떤 기준을 넣을지", "wikilink 매칭 전략을 어떻게 할지". Claude Code는 코드를 잘 쓰지만 "왜 이렇게 해야 하는지"는 사람이 정해줘야 한다.

동시성 문제와 스케줄링 진화는 이 포스트에서 다루기엔 길다. 각각 별도 포스트로 분리했다.

- [동시성편 — Race Condition 6개와 파일 락](/posts/동시성-문제-해결기)
- [스케줄링편 — 디바운스에서 고정 슬롯까지](/posts/스케줄링과-세션-머징)

지금 이 포스트를 쓰는 동안에도 vsc-secondbrain이 돌고 있다. 20분 후에 이 대화도 Obsidian에 저장될 거다.
