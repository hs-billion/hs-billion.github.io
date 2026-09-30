---
layout: post
title: "채팅 LLM이 아니라 소프트웨어의 smart if — TypeSafe Jev"
date: 2026-09-19 16:52:00 +0900
categories: [dev]
permalink: /posts/2026-09-19-typesafe-jev-system-one/
---

TypeSafe AI가 공개한 [System One Models와 첫 모델 Jev](https://typesafe.ai/blog/introducing-system-one-models-and-jev)는, 사람이 읽는 답변을 늘리는 제품이 아니라 **코드 안에 넣는 빠른 구조화 결정**을 목표로 한다. 이 글은 딥러닝 수식 없이, 원문 발표를 개념만 풀어 읽는다.

**초안 작성·문장 정리에 AI를 사용했다.** 해석은 정리자의 것이며 원문과 다를 수 있다.

- 출처(원문): https://typesafe.ai/blog/introducing-system-one-models-and-jev
- 작성: Diogo Almeida (TypeSafe founder)
- 게시(원문): 2026-09-15
- 관련 키워드: system-one · structured-decisions · typed-outputs · routing · guardrails

## Jev가 무엇인가

채팅창에 문장을 길게 써 주는 LLM과 역할이 다르다. Jev는 **소프트웨어가 바로 쓸 수 있는 구조화된 결정**을 내도록 설계된 System One Model의 첫 공개 모델이다. 입력이 비정형 상태(텍스트·프로그램 상태 등)여도, 출력은 미리 정한 스키마의 타입 안전 값이다.[1]

원문이 한 줄로 고정한 비유는 이것이다.

> unstructured state in → typed probabilistic decisions out  
> (프론티어급 지능을 **함수 호출**처럼 쓴다)

즉 “대화 상대”가 아니라, `if` / `switch` / 라우터처럼 코드 경로에 꽂는 **결정 함수**에 가깝다.[1]

## LLM과 무엇이 다른가

원문이 대비표로 정리한 차이를 개념만 추리면 다음과 같다.[1]

| | 기존 LLM | System One + Jev |
|---|---|---|
| 최적화 목표 | 사람이 선호하는 글·채팅, 또는 검증 가능한 생성 | System One 과제에서의 **보정된(calibrated) 결정** |
| 입력 강조점 | 순차 메시지 | 구조화된 프로그램 상태 |
| 출력 | 문자열(유연하지만 파싱·검증·탈선 위험) | **타입 안전** 구조체 + 확률·신뢰도 |
| 샘플링 | 토큰을 하나씩 이어 생성 | **병렬**로 출력을 한 번에 |
| 소프트웨어 관점 | 응답을 파싱해야 코드에 넣음 | 출력이 곧 분기·점수·라벨 |

핵심 트레이드오프는 **문자열 생성을 포기**한 대가다. 자유 문장·장문 작성은 안 하지만, 그 대신 타입 오류 없이 구조화 출력을 내고, 매 답에 확률·신뢰도를 붙이며, 병렬 샘플링으로 속도와 비용 쪽 이점을 노린다. 원문은 가격·지연·벤치마크 수치를 표와 평가 페이지로 공개하지만, 이 글에서는 수치를 재주장하지 않는다. 관심 있으면 원문 표와 workflow evals를 직접 보면 된다.[1]

## 이름: System One과 Jev

**System One**은 Kahneman의 *Thinking, Fast and Slow*에서 말하는 빠른 System 1(직관적 사고)과 느린 System 2(숙고) 구분에서 따왔다. 모델 클래스 이름은 “빠르고 직관적인 결정” 쪽에 가깝다. 원문은 System 1이 곧 오류 투성이라는 함의와 거리를 두며, System One Models를 더 신뢰할 수 있게 만들 수 있다고 본다(세부 논거는 후속 글에 맡김).[1]

**Jev**는 경제학자 William Stanley Jevons에서 왔다. 증기기관 효율이 올라가자 석탄 수요가 오히려 늘었듯, **지능의 비용이 한 자릿수씩 내려가면 쓰임새가 그보다 더 크게 열린다**는 기대를 이름에 담았다.[1]

## 어디에 쓰나

원문이 꼽는 자리는 “사람 옆 코파일럿”보다 **코드 안의 fuzzy decision rule**이다.[1]

- 분류·라우팅·스코어링·추출·분기 — 손코딩 `if`가 너무 깨지기 쉬운 곳
- LLM 프롬프트·추론 흔적·출력에 대한 점수·판정·검증·가드레일·탈옥 감지
- UX가 민감한 **실시간 앱**에서, 구조화 결정을 짧은 지연으로 넣는 경우
- 큰 데이터를 기능·인사이트로 줄이는 map-reduce 성격의 파이프라인

한 줄로 말하면, 제품 코드의 **smart if**다. 주변 코드가 자유도를 제한하므로, 문자열 생성 모델보다 조합·자동화에 넣기 쉽다는 것이 원문의 주장이다.[1]

## 한계 (맞지 않는 곳)

원문 대비를 뒤집으면 경계가 분명하다.[1]

- **자유 문장 생성·장문 작성·채팅 응답** — Jev가 포기한 축이다. 사람용 글·코드 생성·데모용 프로토타입 문장은 기존 LLM 쪽이 맞다.
- **사람이 개입해 읽으며 고치는 작업** — 챗봇·코파일럿·코딩 에이전트처럼 HITL이 전제인 유스케이스는 LLM의 유연성이 강점이다.
- **정답이 싸게 자동 검증되는 생성 루프**(증명·커널 최적화 등) — 원문은 이 축도 LLM의 강점으로 적어 둔다.

정리하면, Jev는 “더 똑똑한 챗봇”이 아니라 **타입과 확률을 가진 결정 API**에 가깝다. 문장을 쓰게 하지 말고, 소프트웨어가 분기할 라벨·점수·경로를 받게 설계할 때 원문의 그림과 맞는다.[1]

## Sources

[1] https://typesafe.ai/blog/introducing-system-one-models-and-jev — Diogo Almeida, Introducing System One Models & Jev (개념·대비표·이름·유스케이스)
