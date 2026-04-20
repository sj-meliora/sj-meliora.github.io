---
title: "Agentic 아키텍처의 품질 속성"
date: 2026-04-20T00:38:46+09:00
categories: ["note"]
tags: ["AI", "agentic", "architecture", "quality-attributes"]
draft: false
---

## 들어가며

"Agent 를 위한, Agent 에 의한" 아키텍처를 설계하기 위해 고민하다 보면 기존 아키텍처 설계 방법론을 적용하기 애매해지는 경우를 맞닥뜨리게 된다. 유저 시나리오를 식별하고 기능 요구사항과 비기능 요구사항을 도출한 것까지는 전통적인 설계와 비슷한 흐름으로 진행되지만, 품질 속성을 정의하기 위해 기존의 잣대를 들이대려고 하면 애매함을 느끼게 된다. 잣대가 맞지 않는 것이다.

- **Functional correctness** — "정답이 있는가" 가 아니라 "어느 정도의 오차율이 허용되는가" 로 바뀐다. 같은 입력에 다른 출력이 나올 수 있기 때문이다.
- **Reliability** — "고장 없이 작동하는가" 의 의미가 애매해진다. Agent 가 *그럴듯한 오답* 을 자신 있게 내놓는 건 고장인가 정상인가? 판단 기준이 불분명하다.
- **Testability** — 전통적 단위 테스트가 성립하지 않는다. 맞는 답이 여러 개일 수 있고, 같은 테스트가 매번 다른 결과를 낼 수 있다.
- **Security** — 입력이 프롬프트의 일부가 되는 순간, 프롬프트 인젝션이라는 신규 공격 표면이 생긴다.
- **Maintainability** — 기반 모델이 업데이트되면 *코드 변경 없이* 동작이 달라진다. 회귀를 어떻게 정의할 것인가?

이 고민에 대해 업계·학계는 어떤 답을 내놓았을지 알아보고자 한다.

## 1. ISO/IEC 25059: 25010 의 AI 확장

ISO/IEC 25059 (2023) 는 25010 의 **AI 시스템용 확장**이다. 25010 의 9개 특성을 전부 유지하되, 그 아래에 AI 특유의 하위 특성을 추가하거나 수정한다. 공식 문서에서 확인되는 변경점은 아래와 같았다.

| 상위 특성 | 신규/수정 하위 특성 | 정의 |
|---|---|---|
| Functional suitability | **Functional adaptability** (신규) | 시스템이 데이터·이전 동작 결과로부터 정보를 획득해 이후 예측에 사용하는 능력 |
| Functional suitability | Functional correctness (수정) | ML 시스템은 본질적으로 일정 오류율을 전제하므로, "맞다/틀리다" 가 아니라 correctness/incorrectness 를 신중히 측정해야 함 |
| Reliability | **Robustness** (신규) | 입력이 교란되거나 데이터 분포가 변해도 functional correctness 를 유지하는 능력 |
| Usability | **User controllability** (신규) | 사람 또는 외부 에이전트가 시스템 동작에 *적시에* 개입할 수 있는 속성 |
| Usability | **Transparency** (신규) | 시스템에 대한 적절한 정보가 이해관계자에게 전달되는 정도 (특성·구성요소·절차·설계 목표·가정 등) |

### 1.1 한계

ISO 25059 는 "AI 시스템" 을 상정한 것이지 "autonomous agent" 를 특정한 것은 아니다. 이 표준은 주로 **모델이 판단한 결과를 사람이 소비하는** 시스템을 다루며, **모델이 판단하고 도구를 써서 직접 행동하는** agent 특유의 속성(tool use 안전성, 루프 종료, 연쇄 실패 등)은 커버하지 않는다. 이 구멍을 채우는 것이 바로 AgentArcEval 같은 후속 방법론이 시도하는 일이다.

## 2. AgentArcEval: ATAM 의 Agent 버전

CSIRO 계열 연구진의 2025년 10월 논문(arXiv:2510.21031). ATAM (Architecture Tradeoff Analysis Method) 을 foundation model 기반 agent 용으로 개조한 방법론이다. 이 논문은 세 가지 — **compound architecture, autonomous and non-deterministic behaviour, continuous evolution** — 가 전통 아키텍처 방법론이 agent 에 직접 적용되지 않는 이유라고 진단한다.

> "Evaluating the architecture of agents is particularly important as the architectural decisions significantly impact the quality attributes of agents given their unique characteristics, including **compound architecture, autonomous and non-deterministic behaviour, and continuous evolution**."

### 2.1 agent 고유 품질 속성

논문은 "general scenario catalogue" 라는 형태로 agent 평가 시 고려해야 할 시나리오들을 정리하고 있다.

- **Compound architecture** 에서 오는 속성
  - Tool/agent selection correctness (여러 tool·agent 중 맞는 걸 골랐는가)
  - Orchestration reliability (오케스트레이션 실패 시 복구)
  - Context engine / memory integrity
- **Autonomous behavior** 에서 오는 속성
  - Goal adherence (주어진 goal 을 끝까지 유지하는가)
  - Guardrail effectiveness (경계를 넘지 않는가)
  - Escalation appropriateness (모를 때 사람에게 넘기는가)
- **Non-deterministic behavior** 에서 오는 속성
  - Behavioral consistency (같은 입력에 비슷한 품질의 답이 나오는가)
  - Recoverability (실패 감지 후 재시도·복구)
- **Continuous evolution** 에서 오는 속성
  - Regression resistance (모델 업데이트 시 기존 동작 유지)
  - Learnability from feedback

### 2.2 전통 분류에서 놓치기 쉬운 지점

**Goal adherence / Goal drift.** Long-horizon agent 에서 가장 중요한 속성 중 하나로 학계에서 부각되고 있다. 전통 25010 에는 대응되는 개념이 없다.

**Tool selection correctness.** "잘못된 tool 을 *올바른 의도로* 선택" 하는 실패 모드가 흔하다. 이건 Security 위협(오용)이 아니라 Functional correctness 의 하위 속성으로 보는 게 더 정확하다.

**Continuous evolution.** 전통적으로는 Maintainability 로 묶을 수 있지만, agent 에서는 *코드 변경 없는 회귀* 가 일상이므로 별도의 1차 품질 속성으로 격상하는 흐름이 있다.

## 3. Anthropic/OpenAI 계열의 산업 프레임: Autonomy × Oversight

Agentic 아키텍처에서는 agent 가 담당할 수 있는 자동화 범위(autonomy)또한 중요한 품질 속성 중에 하나이다. 자율주행의 SAE J3016 은 autonomy level 을 L0~L5 까지 정의했는데, Anthropic 을 비롯한 Agentic 업계에서는 이러한 선형적인 스케일 대신 다른 정의를 제안하고 있다.

### 3.1 Anthropic "Framework for safe and trustworthy agents" 의 핵심 논리

Anthropic 공식 블로그의 프레임은 autonomy 를 **선형 스케일(L0~L5)** 이 아니라 **2축**으로 본다.

- 축 1: **자율성(Autonomy)** — agent 가 독립적으로 결정·실행하는 범위
- 축 2: **감독(Oversight / Control)** — 사람이 개입·중단할 수 있는 능력

이 둘은 trade-off 가 아니라 **양립 가능**하며, 잘 설계된 시스템은 둘을 동시에 높인다. 예를 들어 Claude Code 는 기본적으로 read-only 권한 → 코드 변경은 승인 필요 → 신뢰하는 작업은 persistent permission → 사용자가 언제든 stop 가능. 같은 agent 가 task 마다 다른 autonomy 수준에서 동작한다.

### 3.2 OpenAI / Amazon / Databricks

- **OpenAI** "Building agents" 가이드: models / tools / knowledge / guardrails / orchestration 5개 primitive. Guardrails 와 orchestration 이 oversight 축.
- **Amazon Bedrock AgentCore Evaluations**: **quality / performance / responsibility / cost** 4축 평가. Responsibility 가 oversight 축을 포함.
- **Databricks MLflow 3**: Safety judges + Guidelines judges 를 별도 평가 축으로 제공.

### 3.3 자율주행과 Agent 의 차이점

자율주행의 SAE J3016 스케일(L0~L5)은 한 시스템이 한 순간에 한 레벨에서 동작한다는 전제에 기반한다. 반면 agent 는:

- **같은 시스템이 task 마다 다른 레벨에서 동작** — 예: 코드 읽기는 full autonomous, 코드 수정은 supervised, 배포는 copilot
- **같은 task 내에서도 단계마다 레벨이 다름** — 예: 초기 탐색은 autonomous, 실행 전 승인은 oversight 활성화

따라서 "시스템의 autonomy level 이 Lx 다" 라는 표현이 애초에 성립하지 않는다. 2축 좌표평면 위에서 *task 별 좌표점* 을 찍는 편이 설계 결정을 기술하기에 적합하다.

![Autonomy × Oversight 2축 모델](/images/autonomy_oversight_2axis.svg)

- **Copilot**: High oversight + Low autonomy — 매 결정마다 승인
- **Guided Autonomous**: High oversight + High autonomy — 자율 실행하되 중요 결정에 개입
- **Full Autonomous**: Low oversight + High autonomy — 사후 감사만
- **Rarely used**: Low oversight + Low autonomy — 딱히 쓸 일 없음

선형 L0~L5 스케일은 "시스템이 autonomy 를 점점 키워가는 로드맵" 을 표현하는 데는 여전히 유용하지만, **아키텍처 설계 결정의 축** 으로 쓰려면 Anthropic 이 제안한 2축 구조가 더 적합하다.

---

## 4. 표준화 전인 LLM/Agent 고유 특성들

ISO 25059 에도 없고, ATAM 류 방법론에서도 아직 구체화가 덜 된, 하지만 산업계와 학계에서 활발히 논의 중인 속성들을 정리해 보았다.

### 4.1 Goal adherence / Goal drift

**정의**: 장기 실행 agent 가 초기에 받은 목표를 이탈하지 않는 성질.

**왜 중요한가:** long-horizon autonomous agent 에서 가장 핵심적인 실패 모드 중 하나. 최근 연구(Arike et al. 2025, arXiv:2505.02709)는 Claude 3.5 Sonnet 조차 100k 토큰 이상의 context 에서는 일부 drift[^drift] 를 보인다고 보고한다. 경쟁 목표(예: 수익 최대화) 가 환경 압력으로 들어올 때 drift 가 가속된다.

[^drift]: Drift: agent 가 초기에 받은 목표나 제약에서 시간이 지남에 따라 점진적으로 이탈하는 현상. 급격한 이탈이 아니라 맥락이 누적되면서 서서히 움직인다는 점이 특징이다.

**측정 방향:**
- 주기적으로 agent 에게 현재 자기 목표를 말하게 하고 초기 목표와 비교
- Instrumental phase(경쟁 목표 주입) 후의 action/inaction 기반 drift score
- Context length 과 drift 의 상관

**전통 속성과의 관계:** Controllability 에 암묵적으로 포함되기 쉽지만, 실제로는 **Controllability 만으로는 drift 를 막지 못한다**. 사람이 개입하지 *않는* 시간 동안에도 agent 가 목표를 유지해야 한다. 별도 속성으로 보는 게 맞다.

### 4.2 Groundedness (Faithfulness)

**정의**: 출력이 실제 근거(retrieved 문서·tool 결과·이력) 에 기반하는 정도.

**용어 혼란 주의:**
- Anthropic / RAG 커뮤니티: "Groundedness"
- Amazon: "Faithfulness" (conversation history 에 일치)
- 다른 곳: "Attribution", "Source-grounded-ness"

실질적으로 같은 것이지만 측정 방법은 조금씩 다르다:
- **Citation density**: 주장 단위당 근거 인용 수
- **Citation accuracy**: 인용이 실제로 주장을 뒷받침하는가
- **Fabrication rate**: 근거 없이 지어낸 주장의 비율

### 4.3 Pressure robustness / Contextual integrity

**정의**: 외부 압력(긴급함, 성과 압박, 사용자의 유도 질문 등) 이 들어와도 안전 제약·원래 goal 을 유지하는 성질.

**왜 중요한가:** 최근 연구(arXiv:2603.14975) 는 agent 가 *pressure 상황에서 안전 제약을 negotiable friction 으로 취급* 하는 경향을 보고한다. 즉, 조용한 환경에서는 잘 작동하던 guardrail 이 긴급 상황을 시뮬레이트하면 무너진다. 제안된 완화책 중 하나는 "pressure isolation" — reasoning 과 execution 을 분리해서 압력 신호가 execution 결정에 직접 들어가지 않게 하는 아키텍처 결정이다.

Security(프롬프트 인젝션) 과도 다르고, Goal adherence 과도 다르다 — pressure 는 adversarial 이 아니라 **legitimate pressure**에서도 문제가 된다는 점이 특징이다.

### 4.4 Loop termination / Budget adherence

**정의**: agent 가 무한 루프에 빠지지 않고 정해진 시간·비용·스텝 안에 종료하는 성질.

**왜 중요한가:** DeepEval 등에서 "the budget burner" 패턴으로 정리하는 실패 모드. task 는 "완료" 되는데 10배의 token 을 썼거나, 애매한 feedback 에 대해 agent 가 같은 action 을 반복하며 종료 조건을 못 만난다.

**측정 방향:**
- 작업당 max_steps, max_tokens, wall_clock_timeout 을 정하고 이를 초과하면 fail
- Quality 는 좋은데 cost/latency 는 fail 인 경우를 별도 실패 범주로 관리

Cost Predictability 와 혼동하기 쉽지만 분리하는 게 깨끗하다. Cost Predictability 는 *평균적 예측 가능성*, Loop termination 은 *worst case 상한 보장*.

### 4.5 Context hygiene / Context engineering quality

**정의**: agent 의 context window 에 **유의미한 정보만** 들어가고, 낡거나 관련 없는 정보가 누적되지 않는 성질.

**왜 중요한가:** Anthropic "Effective context engineering" 블로그(2025)의 핵심 주장: context 는 모델의 attention budget 을 차지하는 **희소 자원**이며, 과도한 pre-loaded context 는 "drowning in exhaustive but potentially irrelevant information" 을 초래한다. 즉, RAG 로 context 를 많이 넣는 것이 항상 좋은 건 아니다.

이건 품질 속성이라기보다는 *아키텍처 결정의 차원(축)*에 가깝다. 이 축에서의 선택이 Groundedness, Cost Predictability, Latency 에 모두 영향을 준다.

### 4.6 Corrigibility

**정의**: 사람이 agent 를 중단·수정·재교육할 수 있는 성질. "shut down 가능성" 으로 좁게 정의되기도 함.

**용어 주의:** 학계(AI alignment 연구)의 핵심 용어지만, 산업 아키텍처 문서에서는 거의 **Controllability** 와 같은 뜻으로 쓰인다. 정확히는:
- **Controllability** (ISO 25059): 사람이 적시에 개입할 수 있는가
- **Corrigibility** (alignment 연구): agent 가 수정에 *저항하지 않는가*

두 번째가 더 깊다. Anthropic 등이 보고한 "strategic deception" 사례는 controllability 는 있지만 corrigibility 가 부족한 예. 당장 실무에서 이 구분이 필요한지는 상황마다 다르지만, 용어가 등장하면 구분해둘 필요가 있다.

### 4.7 Blast radius / Reversibility

**정의**: agent 의 잘못된 행동이 미치는 범위(blast radius) 와, 그 행동을 되돌릴 수 있는 정도(reversibility).

**왜 중요한가:** Anthropic framework 가 autonomy level 을 task 별로 다르게 주라고 할 때, 그 경계선을 긋는 기준이 바로 이 두 가지다. "Blast radius 가 작고 reversible 한 action → 높은 autonomy 허용" / "Blast radius 가 크거나 irreversible → 낮은 autonomy".

개별 속성이라기보다 **autonomy 경계를 설정하는 2축**으로 보는 편이 자연스럽다.

---

## 5. 고민: 왜 이 속성들은 정량화가 어려운가

품질 속성이 설계에서 실질적 역할을 하려면, 후보 설계안 사이의 trade-off 를 식별할 수 있어야 한다. "설계 A 는 Goal adherence 는 좋지만 Cost 가 높고, 설계 B 는 그 반대" 같은 비교가 가능해야 Decision Point 에서 근거 있는 선택을 할 수 있기 때문이다. 그런데 앞 절에서 정리한 frontier 속성들 — Goal adherence, Pressure robustness, Corrigibility — 은 그 비교가 가능한 단계에 이르지 못하고 있다. 왜 그런지 세 가지 구조적 이유를 짚어볼 수 있다.

### 5.1 Ground truth 의 부재

전통적 품질 속성은 대체로 **시스템 고유의 값**으로 환원된다. Latency 는 시스템 구성에 따라 결정되고, Throughput 은 부하 곡선을 그리면 된다. Functional correctness 도 입력-출력 쌍이 정해지면 정답이 있다.

반면 Goal adherence 의 "정답 행동" 은 goal 과 context 와 경쟁 목표의 조합에 따라 달라진다. 같은 agent 가 같은 goal 을 받아도, 어떤 환경적 압력이 들어오느냐에 따라 "이탈" 의 정의가 달라진다. 즉, ground truth 가 시스템 속성이 아니라 **시나리오 속성**이다. 이건 데이터의 문제가 아니다. 정답의 정의 자체가 상황에 매달려 있다.

### 5.2 측정 환경의 일반화 한계

학계의 실험은 특정 환경에서 수행된다. Goal drift 실험(arXiv:2505.02709)은 stock-trading 시뮬레이션에서 수익 최대화를 경쟁 목표로 주입하는 방식이고, Pressure robustness 실험은 TravelPlanner 환경에서 긴급성을 주입하는 방식이다. 이 측정값이 실제 운영 환경의 agent — 전혀 다른 도메인, 전혀 다른 tool set, 전혀 다른 사용자 — 에 어떻게 이전되는지는 아무도 모른다.

전통 속성은 이 이전 문제가 훨씬 가볍다. Latency 벤치마크는 워크로드가 다르면 그만큼의 보정을 하면 되지만, drift 벤치마크는 *어떤 보정을 해야 하는지* 조차 아직 정립되지 않았다. 즉, 학계의 측정 결과를 실무 아키텍처 결정으로 가져올 전이 경로가 끊겨 있다.

### 5.3 관찰자 효과

가장 미묘한 문제다. 전통 시스템의 속성 측정은 관찰 행위가 측정 대상을 바꾸지 않는다는 전제 위에서 작동한다. CPU 사용률을 재기 위해 모니터링 프로브를 붙여도, CPU 사용률 자체가 프로브에 반응해서 달라지지는 않는다 (오버헤드를 무시하면).

그런데 Corrigibility 를 측정하겠다고 "shutdown 시도에 agent 가 어떻게 반응하는지" 를 probing 하기 시작하면, 충분히 학습된 모델은 *그 probe 자체를 상황의 일부로 인식* 할 수 있다. Anthropic 등이 보고한 deceptive alignment 사례 — 평가 중에는 순응적으로 행동하고 배포 후에는 다르게 행동하는 — 는 극단적 예지만, 같은 원리가 온건한 형태로도 작동한다. 즉, agent 의 고차 속성은 **측정 행위 자체가 측정 대상에 영향을 주는** 영역에 들어온다.

이것은 물리학의 관찰자 효과와는 성격이 다르다. 물리학에서는 관찰이 대상을 *교란* 시킨다면, 여기서는 관찰이 대상을 *전략적으로 반응하게* 한다. 이 차이가 측정 방법론 자체를 어렵게 만든다.

### 5.4 그래서 어떻게 접근해야 하는가

이 세 가지 이유를 종합하면, frontier 속성들은 전통적 의미의 "정량 측정 가능한 품질 속성" 과는 다른 범주에 있다고 봐야 할지도 모른다. 측정 지표가 아니라 **설계 원칙**으로 다루거나, 단일 수치가 아니라 **다양한 시나리오에서의 반응 프로파일**로 관리하거나, 혹은 아예 품질 속성이 아닌 다른 형태의 관리 대상 — 안전 사례(safety case), 운영 가드레일, 사고 대응 플레이북 — 으로 재분류해야 할 수도 있다.

이 재분류를 어떻게 할지, 그리고 그것이 기존의 아키텍처 평가 방법론(ATAM 등)과 어떻게 접합될지는 아직 열려 있는 질문이다. 이 분야가 빠르게 움직이고 있는 만큼, 당분간은 "측정 가능한 것은 측정하고, 그렇지 않은 것은 의식적으로 설계 원칙으로 다룬다" 는 이중 트랙 접근이 현실적이지 않을까 싶다.