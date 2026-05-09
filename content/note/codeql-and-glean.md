---
title: "CodeQL과 Glean: Semantic Code Index 도구 비교"
date: 2026-05-09T23:54:23+09:00
categories: ["note"]
tags: ["large-codebase", "C++", "agent", "semantic-code-index", "CodeQL", "Glean", "static-analysis"]
draft: false
---

> **TL;DR.** 쿼리형 semantic code index 도구인 CodeQL과 Glean 에 대해서 알아보고, 실제로 프로젝트에 도입하고자 했을 때 검토해야 할 사항들을 정리해 보았다.

---
<!--more-->

## 1. 들어가며

대규모 C++ 코드베이스에 agentic workflow를 적용하기 위한 방편으로, 대표적인 semantic code index 도구인 CodeQL과 Glean에 대해서 알아본다.
비슷한 부류의 도구들 중 이들을 살펴보기로 결정한 이유는 두 도구 모두 2026년 현재 활발하게 개발되고 있으며 C++을 지원하기 때문이다.

## 2. CodeQL

CodeQL은 GitHub의 보안 분석 도구다. 처음부터 *"이 보안 취약점 패턴과 같은 형태를 가진 코드가 또 있는가"* 를
묻기 위해 만들어졌고, 이를 위해 코드베이스를 데이터베이스로 바꾸고 그 위에
QL이라는 쿼리 언어를 얹는 방식을 택했다.

### QL — 쿼리 언어

QL은 SQL과 Datalog의 영향을 받은 declarative 쿼리 언어다. 객체지향 문법으로 표현되며, *어떤 조건을 만족하는 코드 요소가 있는지*를 set 단위로 묻는 결이다. 다음은 C++ 코드에서 `memcpy` 호출 중 세 번째 인자가 상수 0인 경우를 찾는 간단한 예시다.

```ql
import cpp

from FunctionCall call
where
  call.getTarget().hasName("memcpy") and
  call.getArgument(2).getValue() = "0"
select call, "memcpy with size 0"
```

`getTarget()`, `getArgument()`는 단순한 텍스트 매칭이 아니라 컴파일러가 분석한 의미 정보 위에서 동작한다. 매크로 확장, 템플릿 인스턴스화, 오버로드 해소가 모두 반영된 결과를 묻는 셈이다.

### 분석 라이브러리

CodeQL의 가치는 쿼리 언어 자체보다는 함께 제공되는 분석 라이브러리에 있다. 데이터 플로우 분석, taint tracking, 제어 흐름 분석 같은 것들이 모듈 형태로 제공된다. 사용자는 *"어디서 어디로 데이터가 흐르는지"* 같은 질문을, 직접 분석을 작성하지 않고 던질 수 있다.

2026년 들어서는 분석을 declarative하게 확장하는 방향으로 진화하고 있다. `models-as-data` 기능을 통해 사용자가 YAML로 sanitizer나 validator를 정의할 수 있게 됐고, 자체 코드의 검증 로직을 CodeQL이 인식하도록 만드는 작업이 가벼워졌다.

### 최근 동향

개발은 활발하다. 2026년에만 2.24, 2.25 시리즈를 거치며 여러 마이너 릴리스가 나왔고, 3월에는 incremental analysis가 일반 가용성에 도달해 PR 단위 스캔 시간이 크게 줄었다. C/C++ 영역도 false positive 감소와 새로운 쿼리 추가가 꾸준히 이어지고 있다.

## 3. Glean

Glean은 Meta 내부에서 개발된 코드 인덱싱 시스템이다. 2024년 12월에 OSS로 적극적인 푸시가 시작됐다. 보안 분석에서 시작된 CodeQL과 달리, Glean은 Meta의 대규모 모노레포에서 IDE 백엔드와 코드 검색을 뒷받침하기 위해 만들어졌다.

### Angle — 쿼리 언어

Glean은 코드를 *fact* 형태로 추출해 RocksDB에 저장한다 (최근에는 LMDB 백엔드도 실험 중이다). 그 위에 Angle이라는 declarative 쿼리 언어를 얹는다. Angle은 Datalog 계열로, QL보다 단순한 fact-relational 형태다.

다음은 C++ 코드에서 특정 이름의 함수 정의를 찾는 간단한 Angle 쿼리다.

```angle
cxx1.FunctionDeclaration { name = "memcpy" }
```

QL이 *"이런 조건을 만족하는 호출을 찾아라"* 라고 표현한다면, Angle은 *"이런 형태의 fact가 있는가"* 를 묻는다. 표현력은 QL보다 좁은 대신, 인덱싱과 쿼리 모두 가볍게 유지하기 쉽다.

### 스키마와 다언어 지원

Glean의 한 가지 특징은 언어별 스키마를 자유롭게 정의할 수 있다는 점이다. C++의 경우 `using` statement 추적 같은 C++ 특유의 디테일을 스키마에 담을 수 있고, 이를 활용해 미사용 `#include` 같은 dead code 검출도 가능하다. 동시에 언어 중립적인 view도 제공해서 여러 언어가 섞인 코드베이스에 통합 쿼리를 던지는 시나리오도 지원한다.

C++ 인덱서는 clang을 래핑한 형태다. `compile_commands.json`을 입력받고 CMake 빌드를 직접 지원한다. 다만 OSS 릴리스에는 병렬 인덱싱이 빠져 있어서, Meta 내부에서 사용하는 것과 같은 수준의 scale은 외부에서 재현하기 어렵다.

### LSP 백엔드

2026년 2월 릴리스에서 `glean-lsp`가 추가됐다. Glean이 구축한 fact DB 위에 LSP 백엔드를 얹어, VS Code 같은 에디터에서 go-to-definition, find references, hover 같은 표준 LSP 기능을 사용할 수 있게 됐다. 복잡한 쿼리형 질의가 아니라 단순한 탐색 용도로도 쓸 수 있다는 뜻이다. 다만 무게중심은 여전히 fact + Angle 쿼리에 있고, LSP는 부가 인터페이스에 가깝다.

### 성숙도

Glean의 README는 자기 평가가 솔직하다. *pre-release software*, *rough edges*, *limited language indexers*, *the build system is not as smooth as we would like*. 활발하게 개발되고는 있지만, 지금 당장 production에 들이려면 운영 측의 이해와 자체 빌드 통합 작업이 필요하다. Meta 외부의 도입 사례도 아직 많지 않다.

![CodeQL과 Glean 비교 흐름도](/images/codeql-glean-flow.svg)

## 4. 도입 평가

쿼리형 질의 솔루션으로 두 도구를 도입하고자 했을 때 고려해야 할 부분들을 정리해 보았다.

| 평가 항목 | CodeQL | Glean |
|--------|--------|-------|
| 라이센스/비용 | GitHub Advanced Security 또는 Enterprise 계약 필요 | BSD 라이센스, 무료 |
| 빌드 통합 | extractor가 빌드 명령을 감싸 실행 | `compile_commands.json` 입력, CMake 직접 지원 |
| 운영 부담 | GitHub 호스팅이 기본, 자체 운영 부담 낮음 | 인덱싱·저장·쿼리 서버 모두 자체 운영 |
| 성숙도 | Production. 광범위한 사용 사례 | 본인들 표현으로 pre-release |
| 학습 자료 | 공식 문서 + 표준 쿼리 라이브러리 + 외부 자료 풍부 | 문서는 있으나 외부 사례 적음 |

Glean은 CMake 기반 빌드 구조에 편입시키기 용이하고, 무료라는 점이 매력적이지만 사용 사례가 적고 문서가 부족하다는 점, 자체적으로 서버를 운영해야 한다는 점에서 선택하기에 부담스럽다. 반대로 CodeQL 은 운영 부담은 적은 대신 빌드 구조를 바꿔야 하고, 결국에는 비용을 지불해야 한다는 점이 걸린다. 아무리 enterprise 계약을 한다고 해도 결국 코드가 GitHub 인프라 위에 올라간다는 점도 감점 요인이다.

## 5. 마치며

'쿼리형 질의로 찾아내기 어려웠던 복잡한 정보들을 쉽고 빠르게 얻어낼 수 있다' 이전 글을 작성하며 semantic code index 가 개발자들에게 가져다 줄 장미빛 미래를 기대하게 되었다면, 이번 글을 준비하면서는 역시 세상에 쉽게 주어지는 것은 없다는 것을 다시 한 번 깨닫는 계기가 되었다. 

개인적으로는 먼저 적당한 규모의 공개 코드베이스를 GitHub 위에서 운영하면서 CodeQL 을 적용해보고, 그 과정에서 얻는 경험을 바탕으로 실제 코드베이스에 도입할 수 있는 다른 도구들을 알아보는 것이 현실적인 선택이 아닐까 한다. 그런 의미에서
다음 글에서는 한 단계 눈높이를 낮춰, agentic workflow 에 쓰기 좋은 — agent friendly한 — LSP 도구들을 알아보고자 한다.