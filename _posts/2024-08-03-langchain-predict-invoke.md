---
layout: post
title: "LangChain predict() vs. invoke()"
subtitle: "predict() is deprecated"
date: 2024-08-03 22:43:00 +0900
comments: True
---

# predict() vs. invoke()

테디 노트님의 [<랭체인LangChain 노트> - LangChain 한국어 튜토리얼🇰🇷](https://wikidocs.net/book/14314) 을 보고 공부하던 중, chain을 실행할 때 어떨 때는 `invoke()`를 쓰고 어떨때는 `predict()`를 써서 둘의 차이가 무엇인지 찾아보았다.

> Code targeted for removal
> Code that has better alternatives available and will eventually be removed, so there’s only a single way to do things. (e.g., `predict_messages method` in `ChatModels` has been deprecated in favor of `invoke`). [출처](https://python.langchain.com/v0.2/docs/versions/v0_2/deprecations/)

그냥 langchain이 버전업되면서 산재하던 `run()`, `predict()` 등의 호출 메소드를 `invoke()`로 통일하고 다른 메소드들은 deprecated 되었다.

앞으로는 `invoke()`로 통일하는 것이 바람직하겠다.

# invoke()

그런데, `invoke()`를 쓰다보면 객체마다 어떤 input을 넣어야 하는지 헷갈릴 때가 있다. prompt, model이 어떻게 input을 받아 `invoke()` 메소드가 실행되는지 파악할 필요가 있다.

## prompt.invoke()

### 인수가 1개인 경우

```python
prompt = PromptTemplate.from_template("{country}의 수도는 어디입니까?")
prompt.invoke("대한민국")
```

```output
StringPromptValue(text='대한민국의 수도는 어디입니까?')
```

```python
prompt = PromptTemplate.from_template("{country}의 수도는 어디입니까?")
prompt.invoke({"country":"대한민국"})
```

```output
StringPromptValue(text='대한민국의 수도는 어디입니까?')
```

```python
prompt = PromptTemplate.from_template("{country}의 수도는 어디입니까?")
prompt.invoke({"input":"대한민국"})
```

```output
KeyError: "Input to PromptTemplate is missing variables {'country'}. Expected: ['country'] Received: ['input']"
```

입력 인수가 1개면, `input_variables`의 key와 일치하도록 dict로 전달하거나, 그냥 value 값 하나만 전달할 수 있다.

### 인수가 여러 개인 경우

```python
from langchain_core.prompts import PromptTemplate

prompt = PromptTemplate.from_template("{country}의 {info}는 얼마입니까?")
prompt.invoke("대한민국")
```

```output
TypeError: Expected mapping type as input to PromptTemplate. Received <class 'str'>.
```

입력 인수가 2개 이상인 경우, dict type으로 인수를 전달하는 것이 강제된다.

## model.invoke()

```python
model = ChatOpenAI(model='gpt-4-turbo-preview')
model.invoke({"input":"Hi"})
```

```output
ValueError: Invalid input type <class 'dict'>. Must be a PromptValue, str, or list of BaseMessages.
```

model 객체의 invoke는 `str` 또는 `PromptValue`, `BaseMessages`의 list만 입력 인수로 받을 수 있다. 그래서 단순한 경우(단순 `str` 문자열만 전달하는 경우)가 아닌 이상 model의 앞단에 prompt를 연결하게 된다.
