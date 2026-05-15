# Hermes Agent

터미널에서 `hermes`를 입력하면 대화가 시작되고, 파일을 읽고 쓰고, 웹을 검색하고, 명령을 실행하고, 메시지를 보내는 등 다양한 작업을 수행

메모리와 스킬로 사용할수록 똑똑해지는 "성장하는 에이전트"는 OpenClaw를 비롯해 이 분야의 공통 흐름

Hermes Agent도 같은 방향이지만, 복잡한 작업이 끝난 뒤 백그라운드에서 대화를 리뷰하고 스스로 스킬을 생성하는 자동화 메커니즘에 더해, 백그라운드 큐레이터가 7일 주기로 라이브러리 전체를 정리해 주는 단계까지 와 있는 것이 특징

## Architecture

React + Iteration budget + Tool

Self-evaluation System (DPSy + GEPA)

<https://github.com/NousResearch/hermes-agent-self-evolution>

이 별도 시스템은 DSPy + GEPA(Genetic-Pareto Prompt Evolution)를 사용하여 에이전트의 구성요소를 자동으로 최적화 함

## DPSy

DPSy (Declarative Self-improving Language Programs, pythonically)

DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines (The Twelfth International Conference on Learning Representations Journal, 2024)

DSPy를 단순한 라이브러리 소개가 아니라 LLM 애플리케이션을 구축하는 패러다임 자체의 전환으로 제시

파이썬 스타일로 작성된 선언적이고 스스로 개선되는 기능을 갖춘 자연어 처리 프로그램을 의미

이 프레임워크에서는 LLM 파이프라인이 무엇을 할 것인지를 명확히 선언하면, 내부적으로 스스로 학습하고 최적화하여 성능을 향상시키는 기능이 있음

AlixPartners의 기술 컨설턴트 Kevin Madura가 AI Engineer 컨퍼런스에서 발표한 세션

## 문제 정의

### 현재 LLM Application 개발 문제

전형적인 RAG 파이프라인의 프롬프트:

```text
"You are a helpful assistant. Given the context below, answer the question.
 Be concise. If you don't know, say 'I don't know'.
 Context: {context}
 Question: {question}
 Answer:"
```

문제점:

1. 취약성: 단어 하나 바꾸면 성능 급변 ("Be concise" 제거 → 성능 10% 하락)
2. 모델 종속: GPT-4에서 튜닝한 프롬프트가 Llama에서는 작동 안 함
3. 파이프라인 복잡도: RAG = 검색 + 재랭킹 + 생성, 각 단계마다 프롬프트 튜닝 필요
4. 비체계적: 프롬프트 변경의 전체 파이프라인 영향을 예측 불가

→ 핵심 제안: 프롬프트를 직접 쓰는 대신, 무엇을 할지(what)를 선언하고 어떻게 할지(how)는 컴파일러에 맡긴다.

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

### Teleprompter (Optimizer): 자동 최적화

컴파일 과정:

1. 학습 데이터 (질문, 정답) 쌍 준비
2. Teleprompter가 자동으로:
   - Few-shot 예시 선택 (BootstrapFewShot)
   - 또는 instruction 최적화 (COPRO, MIPRO)
   - 또는 파인튜닝 데이터 생성 (BootstrapFinetune)
3. 각 모듈의 프롬프트를 독립적으로 최적화
4. 전체 파이프라인의 end-to-end 메트릭 기반 평가

```python
teleprompter = BootstrapFewShotWithRandomSearch(
    metric=answer_exact_match,
    max_bootstrapped_demos=4,
    num_candidate_programs=16
)
compiled_rag = teleprompter.compile(RAG(), trainset=trainset)
```

## 핵심 철학 : 분리

```text
개발자가 정의하는 것:     컴파일러가 결정하는 것:
  - 입출력 스키마          - 프롬프트 텍스트
  - 모듈 구조              - Few-shot 예시 선택
  - 평가 메트릭            - Instruction 문구
  - 파이프라인 흐름         - 파인튜닝 여부/데이터
```

→ "무엇(what)"과 "어떻게(how)"의 분리

→ 모델을 바꿔도 re-compile만 하면 됨

## 시사점 (영향)

"프롬프트 엔지니어링 → 프롬프트 프로그래밍" 패러다임 전환의 선두주자

LLM 애플리케이션을 소프트웨어 공학적으로 다루는 프레임워크 — 테스트, 최적화, 재현 가능

모델 독립적 개발: 코드 변경 없이 GPT-4 → Llama → Mistral 전환 가능

Stanford NLP 그룹(ColBERT, Alpaca의 저자들)의 집대성

LangChain/LlamaIndex와 다른 철학: 체이닝(how) 대신 선언(what)에 집중

## GEPA

GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning (ICLR 2026 Oral)

## 메모리

## 스킬

## 백그라운드 큐레이터

## Reference

<https://arxiv.org/abs/2310.03714>

<https://github.com/stanfordnlp/dspy>

<https://velog.io/@smj230/%EB%85%BC%EB%AC%B8-%EB%A6%AC%EB%B7%B0-DSPy-Compiling-Declarative-Language-Model-Calls-into-Self-Improving-Pipelines>

<https://arxiv.org/abs/2507.19457>
