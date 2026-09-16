---
layout: post
title: "Text-to-SQL이 막힌 자리 — 분석 에이전트의 문맥 3층"
date: 2026-09-16 23:15:00 +0900
categories: ["Notes"]
permalink: /posts/2026-09-16-claude-code-analysis-agent-context-harness/
---

[田口 信元(@guchey)](https://note.com/guchey/n/n9eb66dd5d470)의 note [Claude Codeで分析エージェントを作って3か月運用した話](https://note.com/guchey/n/n9eb66dd5d470)를 학습·재현에 맞게 다시 구성한 기술 문서다. 출발점은 “SQL을 잘 쓰게 하는 것”이 아니라, **왜**를 묻게 할 문맥을 쌓는 쪽이다.

**초안 작성·문장 정리에 AI를 사용했다.** 해석은 정리자의 것이며 원문과 다를 수 있다.

- 출처(원문): https://note.com/guchey/n/n9eb66dd5d470
- 소개 트윗: https://x.com/koni/status/2100056588568645902
- 작성자: 田口 信元 (@guchey) — Ubie PdM
- 원제: Claude Codeで分析エージェントを作って3か月運用した話
- 게시(원문): 2026-04-04
- 관련 키워드: learning · agents · harness · skills · memory · mcp · llm

## 이 문서가 다루는 문제

제품 KPI·CVR이 흔들릴 때, 원인은 우리 팀 시책일 수도 있고 옆 팀 광고·계절·검색 알고리즘·UI 변경일 수도 있다. 작성자 기준 Ubie에는 **3,000개 이상**의 dbt 모델이 돌아가며, “어느 테이블을 볼지”만으로도 숙련이 필요하다. 이런 판단 지식은 문서화되어 있어도 실상은 **경험값으로 한 사람의 머릿속**에 남는 경우가 많다.[1]

원문이 만든 시스템은 한 줄 질문 — 예: 「프로젝트 A의 CVR이 떨어지는데, 왜?」 — 으로 원인 탐색부터 리포트까지 가게 하려는 분석 에이전트다. 처음은 Text-to-SQL에서 시작했으나 운영 중 벽에 부딪혔다. 필요했던 것은 자연어→SQL 능력 자체보다, **“왜”를 묻기 위한 문맥을 AI에 주는 장치**였다.[1]

대비를 한 줄로 고정하면 다음과 같다.

| | Text-to-SQL에 머무를 때 | 문맥을 쌓을 때 |
|---|---|---|
| 질문 | 숫자를 어떻게 뽑지? | 왜 이런 숫자가 나왔지? |
| 입력 | 스키마·쿼리 능력 | 데이터 밖 사건(시책·릴리스·외부 환경)까지 |
| 실패 모드 | 준 데이터 범위 안에서 그럴듯한 설명 | 교차 소스·날짜 정렬로 가설을 넓힘 |
| 배포 | 개인 프롬프트 | 플러그인·버전관리로 팀 공유 |

## 핵심 개념

### 1. 분석 에이전트의 세 층

원문이 운영에서 정리한 계층은 셋이다.[1]

1. **데이터 층** — 테이블 입도를 맞추고 비즈니스 의미를 모델에 정의한다. “무엇이 일어났는지”를 정확히 잡기 위한 층.
2. **분석 층** — 세그먼트·시계열·이상 감지 등 에이전트의 **행동 규칙**을 정의한다. 탐색은 이 규칙 위에서 돈다.
3. **문맥 층** — 제품 변경·시책 이력처럼 **데이터 바깥**의 사건을 공급한다.

분석 층까지만 있으면 에이전트가 “그럴듯한 거짓말”을 자주 했다고 한다. SEO 데이터만 주면 SEO 안에서, 광고 데이터만 주면 광고 안에서 원인을 찾으려 하고, 실제 주인이 UI 변경이어도 주어진 범위에서 억지로 맞춘다. Text-to-SQL을 “진짜 분석 에이전트”로 올린 결정타가 **제3층(문맥 층)의 팀 횡단 공유**였다고 적는다.[1]

### 2. 탐색적 분석과 에이전트 루프

데이터 분석은 미리 정한 쿼리를 치는 일이 아니라, 1번 결과를 보고 다음을 고르는 시행착오다. Claude Code를 고른 이유로 원문은 이 루프와의 궁합을 든다. SQL 작성 → BigQuery 실행 → 해석 → 다음 액션을 에이전트가 수십 회 돌릴 수 있다는 주장이다.[1]

원문이 든 탐색 예시는 대략 다음 방향전환이다(작성자 사례의 서술).

1. 주간 CV 추이 → 특정 주부터 변동 감지  
2. 채널 분해 → Organic 쪽 변동이 주원인으로 보임  
3. 카테고리 분해 → 특정 카테고리 경향 확인  
4. Search Console 지표 → CTR 변동 포착  
5. 지식에 따라 외부 환경 조사 → 감염증 유행 등과 연결해 리포트  

### 3. 하니스: 모델이 아니라 “무엇을 보여줄지”

원문은 OpenAI의 [Inside OpenAI's in-house data agent](https://openai.com/index/inside-our-in-house-data-agent/)가 말하는 **6층 컨텍스트**를 하니스(모델 능력보다 **모델에 무엇을 보여줄지**를 설계하는 기반)로 소개하고, 자신의 Claude Code 구현을 그에 대응시킨다.[1][2]

OpenAI 측이 공개한 숫자(원문·OpenAI 글 공통으로 인용): 사내 사용자 **3,500명 이상**, 데이터 **600PB** 규모에서 자연어로 물어 수분 내 인사이트를 노린다는 서술이다. 독립 재현 수치가 아니라 공개문의 자체 주장이다.[1][2]

## 작동 원리

질문 한 줄이 들어오면, 에이전트는 고정 대시보드 한 방을 치는 대신 **가설 주도 조사 규칙**을 따른다. 원문이 Agent에 공통으로 둔 규칙은 다음과 같다.[1]

1. **사실 확인** — 먼저 숫자를 본다. 해석하지 않는다.  
2. **가설 발산** — 최소 3개 이상 나열한다. 여기서 좁히지 않는다.  
3. **체계적 검증** — 가설을 하나씩 검증한다. 처음 맞아 떨어진 가설에서 멈추지 않는다.  
4. **반복** — 검증 중 새 사실이 나오면 가설을 다시 만들고 반복한다.  

그럴듯한 첫 가설에서 탐색을 끝내지 않게 막는 장치다.

분석 시점에는 BigQuery 쪽 **메트릭**과 로컬 `memory.md` 쪽 **사업 문맥**을 함께 쓴다. 예: 메트릭이 “어느 주부터 Organic CV 감소”를 보고하고, `memory.md`에 “같은 시기 SNS 유입 시책 종료”, “검색 결과 AI Overviews 표시율 증가”가 있으면 시계열로 정렬해 상관 사건을 고른다. 로그로 직접 이어지지 않아도 **날짜가 맞으면 가설이 된다**는 점이 핵심이라고 한다.[1]

## 구조와 흐름

### Claude Code와 세 층의 맞물림

원문이 정리한 맞물림은 다음과 같다.[1]

- **데이터 층 · 도구 연결** — BigQuery CLI뿐 아니라 Slack·JIRA·Notion·릴리스/커밋·뉴스 검색 등 MCP/CLI로 소스를 넘나든다.  
- **분석 층 · 플러그인 배포** — 스킬 정의를 Markdown으로 커밋하고, 조직 플러그인 마켓플레이스에서 팀이 최신 분석 워크플로를 받는다. 패턴이 생기면 플러그인을 갱신·푸시한다.  
- **문맥 층 · 파일 영속화** — 프로젝트 디렉터리에 분석 과정·결과를 남겨 다음 분석이 자동으로 읽게 한다.

### 플러그인: Entry → Agent → Specialized Skills

KPI 플러그인 골격은 대략 다음과 같다(원문 트리).[1]

```text
kpi 플러그인
├── skills/
│   ├── kpi/SKILL.md           ← Entry: 라우터(직접 분석하지 않음)
│   ├── kpi-product-a/SKILL.md ← Specialized: 제품 A 쿼리군
│   ├── kpi-product-b/SKILL.md
│   └── kpi-release/SKILL.md   ← 릴리스 상관 분석
├── agents/
│   ├── kpi-type-a.md          ← KPI 타입 A 오케스트레이터
│   └── kpi-type-b.md
└── .claude-plugin/plugin.json
```

Specialized Skill의 References가 데이터·문맥 층의 실체다. 원문이 든 네 종류는 다음과 같다.[1]

| 파일 | 역할 |
|---|---|
| `table-knowledge.md` | 이벤트 테이블 정의: 컬럼 의미·타입·입도·JOIN 키·파이프라인 리스크·사용법 |
| `institutional-context.md` | 도메인 지식: CPA/CVR 벤치마크, 캠페인 비교 규칙, CV 정의 차이 등 |
| `learned-corrections.md` | 과거 실패 패턴. 예: 비용만으로 이상 감지하면 CV 트래킹 장애를 놓침 → 비용·CPC·CVR 3축 |
| `queries.md` | 파라미터화된 SQL 템플릿 |

### cron으로 “지금”을 `memory.md`에

시책·릴리스 문맥은 BigQuery에만 있지 않다. 원문은 cron으로 Claude Code를 주기 실행해 MCP/CLI로 Slack·JIRA·Notion·GitHub를 모아 프로젝트별 `memory.md`에 구조화해 쓴다.[1]

`memory.md`에 쌓이는 정보는 두 갈래다.

- **정기 수집** — 시책 보고, 티켓 상태, 릴리스 노트, 클라이언트 피드백 등  
- **분석 후 되쓰기** — KPI 정의, 과거 진단 결론, TODO, 분석 중 발견한 함정  

### OpenAI 6층과의 대응(원문 도표)

원문 도표 「今回の実装との対応関係」가 적은 대응은 대략 다음과 같다. OpenAI 층 이름·설명은 OpenAI 공개문과 원문 도표를 따른다.[1][2]

| OpenAI 층 | 내용(요지) | 원문 구현 |
|---|---|---|
| L1 Table Usage | 스키마·과거 쿼리 패턴 | `table-knowledge.md`, `queries.md` |
| L2 Human Annotations | 도메인 전문가의 테이블/컬럼 설명 | `institutional-context.md` |
| L3 Codex Enrichment | 파이프라인 코드에서 테이블의 참뜻 | 실제 dbt 모델 + `table-knowledge.md`의 파이프라인 리스크 |
| L4 Institutional Knowledge | Slack·Docs·Notion 등 조직 지식 | cron+MCP로 Slack/JIRA/Notion/GitHub → `memory.md` |
| L5 Memory | 수정·학습의 보존 | `learned-corrections.md`, `memory.md`의 분석 수정 이력 |
| L6 Runtime Context | 웨어하우스 실시간 질의 | `bq query`, MCP 실시간 검색 |

## 적용 시나리오

다음은 원문이 서술한 탐색 흐름을, 독자가 필드별로 따라 볼 수 있게 정리한 **원문 기반 시나리오**다(가상 숫자를 새로 만들지 않는다).

- **배경:** PdM이 특정 프로젝트 CVR 하락 원인을 빠르게 가르고 싶다. 지식은 문서와 경험에 흩어져 있다.  
- **목표:** “왜”에 답하는 리포트까지 에이전트가 탐색·교차 검증하게 한다.  
- **입력:** 「프로젝트 A의 CVR이 떨어지는데, 왜?」 수준의 한 줄 질문.  
- **단계:** Entry 스킬이 Agent를 고른다 → Agent가 사실 확인·가설 3개 이상·체계적 검증 루프를 돈다 → Specialized Skill의 References·`memory.md`·BigQuery를 교차한다 → 필요 시 외부 환경 지식을 붙인다.  
- **기대 결과:** 채널·카테고리·외부 요인 등이 묶인 진단 리포트와, 다음을 위한 `memory.md`/교정 파일 갱신.  
- **실패 조건:** 문맥 층이 비어 데이터 범위만으로 설명하려 할 때; 첫 가설에서 조기 종료할 때; 비용 변동만으로 이상을 판정할 때(원문 교정 사례).  
- **검증:** 가설이 3개 이상 검증됐는지, 메트릭 변동과 `memory.md` 사건이 날짜로 대응했는지, 교정 파일이 다음 실행에 읽히는지 확인한다.

## Hands-on

원문 환경(Ubie 내부 BigQuery·조직 플러그인 마켓·실계정)은 그대로 복제할 수 없다. 아래는 **같은 뼈대**를 자기 프로젝트에 옮길 때의 점검·최소 골격이다. 유료 코스·사내 설정 덤프가 아니다.

### 1. 버전

- 런타임: Claude Code를 쓸 수 있는 로컬(또는 팀 표준) 환경  
- 도구: BigQuery CLI(`bq`) 또는 동등한 웨어하우스 CLI, (선택) Slack/JIRA/Notion/GitHub MCP  
- 원문 게시 시점: 2026-04-04. 도구 UI·플러그인 형식은 시점마다 다를 수 있다 — **미검증(환경 의존)**

### 2. 전제조건

- 읽을 수 있는 웨어하우스·최소 하나의 KPI 테이블  
- 시책·릴리스를 적을 수 있는 텍스트 저장소(Git)  
- 외부 도구 연동 시 해당 계정·권한·비밀키(본문에 실제 키를 넣지 말 것)

### 3. 설치

```bash
# 예시: 작업 디렉터리와 플러그인 골격만 만든다 (사내 마켓 URL은 각자 환경)
mkdir -p analysis-agent-plugin/{skills/kpi,agents,.claude-plugin}
mkdir -p analysis-agent-plugin/skills/kpi-product-a
touch analysis-agent-plugin/.claude-plugin/plugin.json
touch analysis-agent-plugin/skills/kpi/SKILL.md
touch analysis-agent-plugin/agents/kpi-type-a.md
```

### 4. 최소 실행

References·memory의 **뼈대 파일**을 만들고, Entry는 라우트만·Agent는 가설 규칙을 갖게 한다.

```bash
cd analysis-agent-plugin
cat > skills/kpi-product-a/table-knowledge.md <<'EOF'
# table-knowledge
- 테이블: <fact_events>
- 입도: <일/세션/유저 중 하나>
- JOIN 키: <user_id, date>
- 리스크: <결측·ARRAY·지연 등 한 줄>
EOF

cat > skills/kpi-product-a/institutional-context.md <<'EOF'
# institutional-context
- CV 정의: <한 줄>
- 벤치마크: <있으면>
- 비교 규칙: <캠페인/기간>
EOF

cat > skills/kpi-product-a/learned-corrections.md <<'EOF'
# learned-corrections
- (예) 이상 감지는 비용·CPC·CVR을 함께 본다
EOF

cat > skills/kpi-product-a/queries.md <<'EOF'
# queries
-- 주간 CV 추이 템플릿 (파라미터만 표시)
-- {{start_date}} ~ {{end_date}}
EOF

cat > memory.md <<'EOF'
# memory
## 정기 수집
- (날짜) 시책/릴리스/이슈 한 줄

## 분석 되쓰기
- (날짜) 진단 결론 / TODO / 함정
EOF
```

Agent 규칙 파일에는 원문의 네 단계(사실 → 가설 3+ → 전수 검증 → 반복)를 그대로 체크리스트로 넣는다.

### 5. 예상 결과

- Entry 스킬이 제품/KPI 타입에 맞는 Agent를 고른다.  
- Agent가 쿼리 결과만으로 결론을 단정하지 않고, `memory.md`와 References를 인용한 가설을 남긴다.  
- 분석 후 `memory.md` 또는 `learned-corrections.md`에 한 줄이라도 되쓰기가 생긴다.

### 6. 검증

```bash
# 뼈대 파일이 있는지
test -f skills/kpi-product-a/table-knowledge.md \
 && test -f skills/kpi-product-a/learned-corrections.md \
 && test -f memory.md \
 && echo "bones ok"

# Agent 규칙에 '가설' 최소 개수 언급이 있는지(문서 검증)
rg -n "가설|hypothesis|3" agents/kpi-type-a.md || echo "rules missing"
```

실쿼리·실MCP 호출은 계정·비용이 필요하므로 이 문서에서는 실행하지 않았다 — **미검증**.

### 7. 대표 오류

- **증상:** 채널/광고 데이터만으로 “원인 확정” 리포트가 나온다.  
  **원인:** 문맥 층·`memory.md`가 비었거나 Agent가 첫 가설에서 종료.  
  **해결:** 시책/릴리스 수집 cron을 켜고, 가설 3개 이상·조기 종료 금지를 규칙에 고정한다.  
- **증상:** 비용은 크게 안 움직였는데 CVR만 붕괴한 장애를 놓친다.  
  **원인:** 이상 감지를 비용 축만으로 수행(원문 교정 사례: CVR 51%→5%, 비용 −34%로 임계 미달).  
  **해결:** `learned-corrections.md`에 다중 축 규칙을 남기고 Specialized Skill이 항상 읽게 한다.[1]

## 읽을 때 경계

- 3개월 운용·반나절→수분·오류 감소 등은 **작성자 운용 경험**이다. 일반 성공률·생산성 배수를 주장하지 않는다.  
- OpenAI 6층·3.5k 사용자·600PB는 OpenAI 공개문의 서술이며, 원문 도표의 대응은 작성자 매핑이다.[1][2]  
- Ubie·조직 플러그인·실데이터 스키마는 공개되지 않는다. Hands-on은 뼈대만 남긴다.  
- 분석 에이전트는 분석 리터러시를 **대체하지 않는다**. 원문도 출력 해석·심화 판단에 도메인·분석 문해가 필요하다고 적는다.[1]  
- Claude Code·MCP·플러그인 마켓 형식은 제품 시점에 따라 달라질 수 있다.

## 남는 한 줄

Text-to-SQL은 출발점이고, **왜**에 답하려면 데이터 밖의 문맥을 쌓아 가설을 돌리며, 그 장치를 플러그인으로 팀 전체에 배포하는 쪽이 원문의 중심이다. 리터러시는 그 위에 남는다.

## Sources

[1] https://note.com/guchey/n/n9eb66dd5d470 — 田口 信元, Claude Code 분석 에이전트 3개월 운용기 (세 층, 플러그인, memory.md, OpenAI 대응 도표, 한계)  
[2] https://openai.com/index/inside-our-in-house-data-agent/ — OpenAI in-house data agent, 6층 컨텍스트·하니스 서술  
[3] https://zenn.dev/ubie_dev/articles/1fe0b284b6173c — 원문이 링크한 Ubie 데이터 분석 기반 소개  
[4] https://x.com/koni/status/2100056588568645902 — 원문 소개 트윗 (@koni)
