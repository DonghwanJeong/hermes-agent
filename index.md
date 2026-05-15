# Hermes Agent

터미널에서 `hermes`를 입력하면 대화가 시작되고, 파일 읽기/쓰기, 웹 검색, 명령 실행, 메시지 전송 등 다양한 작업을 수행한다.

메모리와 스킬을 통해 사용할수록 똑똑해지는 "성장하는 에이전트"는 OpenClaw를 비롯해 이 분야의 공통 흐름이다.

Hermes Agent도 같은 방향을 지향하지만, 복잡한 작업이 끝난 뒤 백그라운드에서 대화를 리뷰하고 스스로 스킬을 생성하는 자동화 메커니즘에 더해, 백그라운드 큐레이터가 7일 주기로 라이브러리 전체를 정리해 주는 단계까지 와 있는 것이 특징이다.

## Architecture

- React + Iteration budget + Tool
- Self-evaluation System: DSPy + GEPA

## Self-evaluation System

별도 시스템인 [hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution)은 DSPy + GEPA(Genetic-Pareto Prompt Evolution)를 사용하여 에이전트의 구성요소를 자동으로 최적화한다.

## DSPy

DSPy는 `Declarative Self-improving Language Programs`의 약자로, 파이썬 스타일로 작성된 선언적이고 스스로 개선되는 자연어 처리 프로그램을 의미한다.

관련 논문:

- `DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines`
- The Twelfth International Conference on Learning Representations Journal, 2024

DSPy는 단순한 라이브러리 소개가 아니라, LLM 애플리케이션을 구축하는 패러다임 자체의 전환으로 제시된다.

이 프레임워크에서는 LLM 파이프라인이 무엇을 할 것인지를 명확히 선언하면, 내부적으로 스스로 학습하고 최적화하여 성능을 향상시킨다.

AlixPartners의 기술 컨설턴트 Kevin Madura가 AI Engineer 컨퍼런스에서 발표한 세션에서도 이 관점이 다뤄졌다.

## 문제 정의

### 현재 LLM Application 개발 문제

전형적인 RAG 파이프라인의 프롬프트:

```text
You are a helpful assistant. Given the context below, answer the question.
Be concise. If you don't know, say 'I don't know'.

Context: {context}
Question: {question}
Answer:
```

문제점:

1. 취약성: 단어 하나만 바꿔도 성능이 급변한다. 예를 들어 `"Be concise"`를 제거하면 성능이 10% 하락할 수 있다.
2. 모델 종속: GPT-4에서 튜닝한 프롬프트가 Llama에서는 작동하지 않을 수 있다.
3. 파이프라인 복잡도: RAG는 검색 + 재랭킹 + 생성으로 구성되며, 각 단계마다 프롬프트 튜닝이 필요하다.
4. 비체계성: 프롬프트 변경이 전체 파이프라인에 미치는 영향을 예측하기 어렵다.

핵심 제안:

> 프롬프트를 직접 쓰는 대신, 무엇을 할지(what)를 선언하고 어떻게 할지(how)는 컴파일러에 맡긴다.

## 제안 방법

### Signature: 입출력 선언

```python
# 기존: 장황한 프롬프트 직접 작성
prompt = "Given a question, search for relevant passages and provide a concise answer..."

# DSPy: 입출력만 선언
class GenerateAnswer(dspy.Signature):
    """Answer questions with short factoid answers."""

    context = dspy.InputField(desc="relevant passages")
    question = dspy.InputField()
    answer = dspy.OutputField(desc="often between 1 and 5 words")
```

### Module: 재사용 가능한 LM 호출 패턴

```python
# 내장 모듈
dspy.Predict(signature)      # 기본 LM 호출
dspy.ChainOfThought(sig)     # 자동으로 "reasoning" 단계 추가
dspy.ReAct(sig, tools=[...]) # ReAct 패턴 자동 구현
dspy.Retrieve(k=3)           # 검색 모듈

# RAG 파이프라인 정의
class RAG(dspy.Module):
    def __init__(self):
        self.retrieve = dspy.Retrieve(k=3)
        self.generate = dspy.ChainOfThought(GenerateAnswer)

    def forward(self, question):
        context = self.retrieve(question).passages
        return self.generate(context=context, question=question)
```

### Teleprompter / Optimizer: 자동 최적화

컴파일 과정:

1. 학습 데이터 `(질문, 정답)` 쌍을 준비한다.
2. Teleprompter가 자동으로 Few-shot 예시를 선택하거나, instruction을 최적화하거나, 파인튜닝 데이터를 생성한다.
3. 각 모듈의 프롬프트를 독립적으로 최적화한다.
4. 전체 파이프라인을 end-to-end 메트릭 기반으로 평가한다.

대표 방식:

- Few-shot 예시 선택: `BootstrapFewShot`
- Instruction 최적화: `COPRO`, `MIPRO`
- 파인튜닝 데이터 생성: `BootstrapFinetune`

```python
teleprompter = BootstrapFewShotWithRandomSearch(
    metric=answer_exact_match,
    max_bootstrapped_demos=4,
    num_candidate_programs=16,
)

compiled_rag = teleprompter.compile(RAG(), trainset=trainset)
```

## 핵심 철학: 분리

| 개발자가 정의하는 것 | 컴파일러가 결정하는 것 |
| --- | --- |
| 입출력 스키마 | 프롬프트 텍스트 |
| 모듈 구조 | Few-shot 예시 선택 |
| 평가 메트릭 | Instruction 문구 |
| 파이프라인 흐름 | 파인튜닝 여부/데이터 |

핵심은 "무엇(what)"과 "어떻게(how)"의 분리다.

모델을 바꾸더라도 코드를 다시 작성하는 것이 아니라, re-compile만 하면 된다.

## 시사점

- "프롬프트 엔지니어링"에서 "프롬프트 프로그래밍"으로 넘어가는 패러다임 전환의 선두주자다.
- LLM 애플리케이션을 테스트, 최적화, 재현 가능성 관점에서 소프트웨어 공학적으로 다루는 프레임워크다.
- 모델 독립적 개발을 가능하게 한다. 코드 변경 없이 GPT-4, Llama, Mistral 등으로 전환할 수 있다.
- Stanford NLP 그룹의 ColBERT, Alpaca 등과 연결되는 연구 흐름의 집대성으로 볼 수 있다.
- LangChain/LlamaIndex와는 철학이 다르다. 체이닝(how)보다 선언(what)에 집중한다.

## GEPA

GEPA는 `Genetic-Pareto Prompt Evolution`의 약자다.

관련 논문:

- `GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning`
- ICLR 2026 Oral

## 메모리

## 스킬

## 백그라운드 큐레이터

## Reference

- <https://arxiv.org/abs/2310.03714>
- <https://github.com/stanfordnlp/dspy>
- <https://velog.io/@smj230/%EB%85%BC%EB%AC%B8-%EB%A6%AC%EB%B7%B0-DSPy-Compiling-Declarative-Language-Model-Calls-into-Self-Improving-Pipelines>
- <https://arxiv.org/abs/2507.19457>
