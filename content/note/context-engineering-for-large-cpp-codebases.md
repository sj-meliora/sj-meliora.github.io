---
title: "대규모 C++ 코드베이스에 Agent 붙이기: Context Engineering 최신 동향"
date: 2026-04-20T23:37:25+09:00
categories: ["note"]
tags: ["large-codebase", "C++", "agent", "firmware", "context-engineering"]
draft: false
---

> **TL;DR.** 300만 라인급 C++ 코드(스토리지 F/W 같은)에 agent를 붙이려면 프롬프트
> 튜닝이 아니라 **"agent가 뭘 보게 할지"를 설계**하는 것이 관건이다. 이 문서는
> 최근 1년간 정리된 세 축을 요약하고, 스토리지 F/W 맥락에서의 함의를 짚는다.
>
> | 축 | 대상 | 무엇을 해결하나 |
> |---|---|---|
> | **Structural** (LSP) | 코드 자체 | "이 symbol이 어디에 쓰이나" — 정적 사실 |
> | **Codified** | 코드 밖 지식 | "왜 이렇게 짰나" — 설계 의도 |
> | **Workflow** (FIC) | context 관리 | "어떤 순서로 보여줄까" — 시간축 |
>
> 세 축은 보완 관계다. 하나만 잘해선 부족하다.

---

## 왜 이 주제인가

300만 라인급 C++ 프로젝트에 agentic workflow를 적용하려고 할 때 가장 먼저
부딪히는 벽은 **agent가 코드를 이해하는 속도**다. 일반 LLM에게 이 규모의 코드는
그냥 "끝없이 열리는 텍스트 파일 더미"다. grep으로 찍어봐야 금방 context가
터지고, 전체를 한 번 훑는 건 애초에 불가능하다.

그래서 최근 1년 사이 "context engineering"이라는 이름으로 정리되고 있는 흐름이
있다. 요점은 단순하다 — **agent가 뭘 보게 할지를 설계하는 것 자체가 별도의
엔지니어링 영역**이라는 것.

---

## 축 1: Structural Context — agent에게 IDE의 눈을 달아주기

### 문제

일반 coding agent는 파일을 텍스트로 읽는다. `OrderProcessor`라는 클래스를
바꾸려면 grep으로 찾고, 쫓아다니면서 관련 파일을 읽고, 그러면서 context window를
다 써버린다. 20파일짜리 프로젝트에선 이게 통하지만 2,000 파일짜리 monorepo에선
금방 막힌다.

### 해법: LSP(Language Server Protocol) 통합

LSP는 원래 IDE용으로 만들어진 프로토콜인데, 이게 agent에게 주는 가치가 예상보다
크다. 핵심은 **정적 분석 결과를 미리 인덱싱해두고 질의한다**는 것. 함수의 모든
call site를 찾거나 symbol 정의로 점프하는 작업이 텍스트 재귀 검색과는 비교할 수
없을 만큼 빠르고 정확하다.

속도가 빨라지면 agent의 **행동 자체**가 달라진다. "비싼" 탐색일 때는 agent가
추측으로 때우려 하는데, "싼" 탐색일 때는 확신이 설 때까지 돌아다닌다. 이게
context window 효율에 큰 영향을 준다 — 잘못된 파일을 여러 번 읽으며 context를
태우는 대신, 맞는 파일에 한 번에 도달한다.

### 대표 구현: Serena (MCP server)

오픈소스로 나와 있는 Serena는 LSP 위에 agent-friendly 추상화를 씌워서 symbol
단위 조작 tool을 MCP로 노출한다. Claude Code, Copilot CLI 같은 agent에서 바로
붙여 쓸 수 있다.

프로젝트 공식 문서에 따르면 서로 다른 agent들이 독립적으로 "이게 있어야
제대로 일한다"고 보고한다. **symbol-aware navigation, cross-file refactor,
monorepo dependency jump**가 근본적으로 달라진다는 것이 공통된 피드백이다.

### 대규모 스토리지 F/W 맥락에서의 함의

clangd 자체는 성숙해 있지만 300만 라인 + 매크로 헤비 + 빌드 variant가 많은
코드에서는 **`compile_commands.json`을 제대로 뽑는 것 자체가 프로젝트**다.
이게 안 되면 LSP는 무용지물이다.

현실적으로 초기 투자의 상당 부분이 "빌드 시스템에서 compilation database를
안정적으로 export하는 파이프라인 정비"에 들어갈 거라고 봐야 한다. Bear, compdb,
CMake export 같은 도구의 조합으로 variant별 DB를 뽑고, agent가 variant를
선택해서 질의할 수 있게 해주는 layer가 필요하다.

---

## 축 2: Codified Context — 코드에 없는 지식을 코드화하기

### 문제

LSP는 "코드에 있는 것"만 본다. 하지만 대규모 프로젝트의 진짜 난이도는 **코드에
없는 지식**에 있다. 왜 이렇게 짰는지, 무엇을 건드리면 안 되는지, 이 매직 넘버가
무슨 의미인지. 이건 아무리 LSP를 잘 써도 안 나온다.

사람들은 흔히 이걸 RAG로 풀려고 한다. 그런데 embedding 기반 retrieval은 **코드를**
indexing할 뿐이다. 필요한 건 **코드에 관한 지식**을 indexing하는 것이다 —
design intent, constraints, failure modes 같은 것들.

### 해법: Codified context infrastructure

2026년 초 arXiv에 올라온 논문 ["Codified Context: Infrastructure for AI Agents
in a Complex Codebase"](https://arxiv.org/abs/2602.20478)가 실용적 구조를
제안한다. 단일 개발자가 108,000라인 C# 분산 시스템을 구축한 경험 기반이고,
세 층으로 구성된다:

1. **Hot-memory constitution**: 항상 로딩되는 convention, retrieval hook,
   orchestration protocol. Claude Code의 `CLAUDE.md`가 여기 해당하지만
   그것만으론 부족하고 훨씬 더 체계적으로 구성된다.
2. **Specialized domain-expert agents**: 영역별로 분화된 subagent들 (이
   케이스에선 19개). 각자 자기 영역의 context만 들고 일한다.
3. **Cold-memory knowledge base**: on-demand로 로딩되는 specification 문서들
   (이 케이스에선 34개). 필요할 때만 꺼내 쓴다.

논문의 핵심 주장은 **"1,000 라인 prototype은 한 프롬프트에 다 담을 수 있지만,
100K 라인 시스템에서는 single-file manifest(CLAUDE.md, .cursorrules)로는
안 된다"**는 것. 300만 라인이면 말할 것도 없다.

### 같은 자산의 두 소비자 — 설계 앞단계와의 연결

> 코드를 짜기 전 단계 — user scenario, 요구사항(FR/NFR), 품질속성, DP(Design
> Point), 설계 결정 — 에서 생성되는 자산은 원래 **사람이 설계 과정을 이해하고
> 개입하기 위해** 만들어진다. 왜 이 구조를 택했는지, 어떤 제약이 있었는지,
> 어떤 대안을 검토했는지.
>
> 그런데 **agent가 코드를 이해하기 위해 필요한 cold-memory**가 정확히 이
> 자산과 겹친다. 코드 자체에는 없는 "왜"에 대한 답이 여기 있기 때문이다.
>
> - 사람 관점: 앞단계 자산 = 설계 결정을 찾고 동기화하기 위한 시스템
> - Agent 관점: 앞단계 자산 = 코드를 이해하기 위한 cold-memory
>
> **즉, 앞단계에서 만드는 자산이 그대로 agent의 cold-memory가 된다.** 두
> 작업이 별개 프로젝트가 아니라 **같은 자산의 두 소비자**다. 이건 투자 정당화에
> 유리하고, 자산 설계 단계에서 "agent도 읽을 수 있는 형식"을 의식하는
> 것만으로도 장기적 가치가 크게 달라진다.

---

## 축 3: Workflow Context — Frequent Intentional Compaction

### 문제

Context window는 크기가 아무리 커져도 **utilization이 높아질수록 품질이
떨어진다.** 파일 작업, 코드 분석, edit history, test log, tool output이 쌓일수록
agent의 추론 품질이 저하된다. 그냥 때려 박으면 중요한 정보가 노이즈에 묻힌다.

### 해법: humanlayer의 FIC 방법론

300K 라인 Rust 코드(BAML) 케이스에서 검증된 방법론이다. 저자는 해당 코드베이스를
처음 접하는 Rust 아마추어였는데, 이 방법론으로 전문가 리뷰를 통과하는 변경을
만들었다고 한다.

**핵심 원칙**: Context 사용률을 **40-60%** 범위로 유지. 이를 위해 워크플로우
자체를 세 단계로 쪼갠다.

```
Research → Plan → Implement
   ↓        ↓         ↓
research.md  plan.md  (code + progress.md)
  (~200줄)  (~200줄)
```

각 단계는 **compacted artifact(구조화된 마크다운)를 산출**하고, 다음 단계는
그 artifact만 입력으로 받는다. 즉, Plan phase는 Research phase의 원본 탐색
과정을 몰라도 된다. 200줄짜리 research.md만 있으면 된다.

### 사람이 개입하는 지점이 바뀐다

이 방법론에서 가장 흥미로운 부분은 **human review의 위치**다.

- 기존 방식: 2000줄 PR을 리뷰 → 이미 다 짜진 뒤라 수정 비용 큼
- FIC 방식: 200줄 research + 200줄 plan을 리뷰 → 실수의 영향이 10-100배 큰
  지점에서 잡음

"이 설계 방향이 맞나"를 플랜 단계에서 200줄 읽고 판단하는 것과, 다 짜진 코드
2000줄을 읽고 뒤늦게 "아 이거 접근 자체가 틀렸네"를 깨닫는 것의 차이다.

### Subagent 설계 교훈

humanlayer 자료와 OpenDev 논문(arXiv 2603.05344) 등에서 공통적으로 나오는 패턴:

- Subagent는 **노이즈가 큰 작업(파일 검색, 코드 분석, 요약)을 격리된 context
  window에서 수행**하고 요약본만 상위로 돌려준다. 이게 없으면 탐색 작업
  하나로 상위 context 사용률이 80%를 넘어버린다.
- Exploration subagent는 **write 권한이 없어야** 한다. OpenDev의 Planner
  subagent는 아예 tool schema에서 write tool을 제거해서 LLM이 write 시도
  자체를 할 수 없게 만든다. Blast radius 축소.
- Task management tool은 main agent만 가진다. Subagent가 todo를 건드리면 race
  condition.

### 스토리지 F/W 맥락에서의 함의

이 패턴은 **스토리지 F/W 개발의 코어 워크플로우(빌드, 검증)와 자연스럽게 맞닿는다.**
FIC의 Implement phase는 "plan.md를 입력으로 코드 변경 → 검증 루프"인데, 이
검증 루프를 코어 워크플로우가 이미 좋은 UX로 묶어주고 있다. FIC를 붙이려면
코어를 재설계할 필요가 없고, 코어 **앞**에 Research-Plan 단계를 붙이면 된다.

---

## 세 축은 어떻게 얽히는가

세 축은 서로 다른 층위의 문제를 풀고, 서로 보완한다.

LSP 없이 FIC만 하면 Research phase가 grep 뺑뺑이로 context를 태운다.
Codified 없이 LSP만 쓰면 agent가 "왜?"를 영영 모른다. 셋 다 없이 그냥
프롬프트만 잘 짜면 1만 라인부터 무너진다.

특히 codified context는 **축 1과 축 3 사이의 접착제** 역할도 한다.
LSP가 찾아준 symbol에 대한 설계 의도(codified)가 있으면 Research phase가
훨씬 짧게 끝난다.

---

## 스토리지 F/W에서 추가 설계가 필요한 지점들

스토리지 F/W 개발 시에 좀 더 고민이 필요한 부분들을 정리해 보았다.

### (a) HW-종속 semantic이 LSP 바깥에 있다

레지스터 맵, 타이밍 제약, 전원/클럭 시퀀스 — HW 동작을 제어하기 위한 코드들은 semantic 을 코드만 보고 이해하기 어렵다. `0x1C` 같은 매직
넘버가 등장하는 경우도 수없이 많다. clangd는 이게 "특정 하드웨어의 일부 기능을 Enable 하기 위한 특수 레지스터"라는 걸 모른다.

이렇게 코드만 봐서는 이해하기 어려운 semantic 을 codified context 로 보완해야 하는데, HW 동작을 어떻게 추상화해서 어떤 형식으로 기술할지 고민이 필요하다.

### (b) "테스트 통과"가 약한 신호다

일반 소프트웨어는 단위 테스트가 견고한 feedback loop을 만들어준다. 펌웨어 개발에서도 단위 테스트는 사용할 수 있지만, 예상치 못한 race나 타이밍 이슈가 발생할 수 있는 FPGA, 실장 테스트에서는 결정론적 재현이 어려워 강한 feedback 을 주기는 어렵다. "정말 통과한 것 맞아? 아무 문제 없는 것 맞아?" 를 확신하기 어렵다는 뜻이다.  

Implement phase의 feedback이 약해지면 **세 단계 분리의 효용이 줄어든다.**
Plan이 아무리 잘 짜여 있어도 실행 결과가 "일단 돌아는 간다" 정도로만 확인되면,
FIC가 가정하는 "빠른 검증-수정 루프"가 성립하지 않는다. 이 프로젝트의 검증
워크플로우가 얼마나 강한가가 FIC 도입의 성패를 가른다.

### (c) 매크로와 conditional compilation

같은 함수가 variant별로 다르게 컴파일된다. LSP는 "현재 active한 컴파일 조건"에서
만 semantic을 제공한다. Agent가 "이 변경이 어느 variant에 영향을 미치나"를
이해하려면 LSP만으로는 부족하고, **variant matrix를 명시한 별도의 context
layer**가 필요하다.

### (d) 높은 도메인 지식 밀도

스토리지 F/W는 일반 SW보다 **라인당 암묵적 지식의 밀도가 훨씬 높다.** FTL, GC, wear
leveling, 에러 처리 — 각 함수에 수년치 시행착오가 녹아 있다. 이걸 codified
context로 어떻게 뽑아낼 수 있을까? 그 자체로 도전적인 목표이지만, workflow 뒷단에서 필요로 하는 불량 분석·재발 방지 단계에도 활용될 가능성이 높아 반드시 달성해야 할 목표이기도 하다.

---

## 관련 자료

### 필독
- [humanlayer/advanced-context-engineering-for-coding-agents](https://github.com/humanlayer/advanced-context-engineering-for-coding-agents) — FIC 방법론. 300K Rust 케이스 포함.
- [Codified Context 논문 (arXiv 2602.20478, 2026)](https://arxiv.org/abs/2602.20478) — Hot/Cold memory 구조 제안.
- [Martin Fowler, "Context Engineering for Coding Agents" (Thoughtworks Exploring Gen AI)](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html) — 전반적 개괄.

### 도구
- [Serena](https://github.com/oraios/serena) — LSP + MCP. C++ 포함 다언어 지원.

### 보조 레퍼런스
- [OpenDev 논문 (arXiv 2603.05344)](https://arxiv.org/html/2603.05344v1) — Subagent 설계, 격리된 context window, read-only 강제.
- [humanlayer/Codified Context 참조 구현](https://github.com/arisvas4/codified-context-infrastructure) — 논문의 3-tier 구조 레퍼런스.

---

## 다음으로 알아볼 만한 것들

1. **Serena 실제 붙여보기**: 우리 빌드 시스템에서 compile_commands.json 뽑고
   clangd 통해 Serena 돌려보기. 가장 빠른 PoC.
2. **Codified context의 최소 단위 실험**: 가장 자주 건드리는 모듈 하나 골라서
   spec 문서를 수동으로 만들어보고, 있을 때/없을 때 agent 성능 차이 보기.
3. **앞단계 지식 시스템의 자산 포맷 설계**: agent 친화적 포맷을 처음부터
   의식하면서. 이게 가장 장기 레버리지가 크다.
