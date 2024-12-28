---
layout: post
title: "LangChain 라이브러리로 시작하는 LLM과 Prompt Engineering"
subtitle: "LangChain을 이용한 LLM 활용 가이드"
date: 2024-08-03 22:43:00 +0900
comments: True
---

<head>
  <style>
    table.dataframe {
      white-space: normal;
      width: 100%;
      height: 240px;
      display: block;
      overflow: auto;
      font-family: Arial, sans-serif;
      font-size: 0.9rem;
      line-height: 20px;
      text-align: center;
      border: 0px !important;
    }

    table.dataframe th {
      text-align: center;
      font-weight: bold;
      padding: 8px;
    }

    table.dataframe td {
      text-align: center;
      padding: 8px;
    }

    table.dataframe tr:hover {
      background: #b8d1f3;
    }

    .output_prompt {
      overflow: auto;
      font-size: 0.9rem;
      line-height: 1.45;
      border-radius: 0.3rem;
      -webkit-overflow-scrolling: touch;
      padding: 0.8rem;
      margin-top: 0;
      margin-bottom: 15px;
      font: 1rem Consolas, "Liberation Mono", Menlo, Courier, monospace;
      color: $code-text-color;
      border: solid 1px $border-color;
      border-radius: 0.3rem;
      word-break: normal;
      white-space: pre;
    }

.dataframe tbody tr th:only-of-type {
vertical-align: middle;
}

.dataframe tbody tr th {
vertical-align: top;
}

.dataframe thead th {
text-align: center !important;
padding: 8px;
}

.page\_\_content p {
margin: 0 0 1.3rem !important;
}

.page\_\_content li > p {
margin: 0 0 0.6rem !important;
}

.page\_\_content p > strong {
font-size: 1.0rem !important;
}

  </style>
</head>

**주요내용**

-   📚 LangChain 라이브러리 설치 및 설정 방법
-   🔑 API_KEY 환경변수 설정으로 편리한 LLM 사용
-   🛠 Runnable 객체를 활용한 chain 구성과 활용법

# LangChain 공부하기

테디 노트님의 [<랭체인LangChain 노트> - LangChain 한국어 튜토리얼🇰🇷](https://wikidocs.net/book/14314) 를 보면서 LangChain을 공부하고 있다. Prompt Engineering에 대해 간단하게 공부를 해보니 이를 직접 적용하여 LLM을 활용하고 싶어졌고, 이를 위해서는 LangChain이라는 라이브러리를 익힐 필요가 있다고 생각했다.

확실히 만들어진지 얼마 안 된 라이브러리이니만큼 오류도 많고, 수정도 실시간으로 이루어진다. 구글링을 해봐도 도대체 원인을 알 수가 없어서 LangChain Github Repo를 들어가보나 13시간 전에 오류가 수정되어 있다든가... 여하튼 공부도 재밌지만 이런 초창기 프로젝트인 라이브러리를 이용해보는 경험 자체가 신비롭다.

이런 개발 관련 블로그는 Ipython Notebook으로 작성하고 적당히 다듬은 후 (이것도 LLM으로 가능!) 올리는게 꽤 괜찮은 방법인 것 같다. 마침 테디 노트님이 해당 방법론에 대한 가이드도 만들어주셔서 보고 적용해보려 한다.

## 기본 세팅

`langchain`을 `pip`로 설치해주기만 하면 된다. `langsmith` 설정은 나중에 필요성을 느낄 때 해도 늦지 않을 것 같아서 생략했다.

LLM을 사용하기 위해서는 각 서비스에서 제공하는 API_KEY를 발급 받아야 한다. 이를 환경변수에 저장해두면 따로 파라미터로 전달하지 않고 편리하게 사용할 수 있다.

테디노트에서는 `dotenv`를 사용하는데, 나는 그냥 귀찮아서 `.zprofile`에 (zsh 기준)에 등록했다. 주요 서비스 별 key는 다음과 같다.

-   OpenAI: `OPENAI_API_KEY`

-   Google Gemini: `GOOGLE_API_KEY`

```python
%%capture
%pip install langchain
```

```python
from IPython.display import display, Markdown
```

```python
from langchain_openai import ChatOpenAI

# 객체 생성
llm = ChatOpenAI(
    temperature=0.1,  # 창의성 (0.0 ~ 2.0)
    model_name="gpt-4o-mini",  # 모델명
)

# 질의내용
question = "대한민국의 수도는 어디인가요?"

# 질의
print(f"[답변]: {llm.invoke(question)}")
```

```
[답변]: content='대한민국의 수도는 서울입니다.' response_metadata={'token_usage': {'completion_tokens': 8, 'prompt_tokens': 16, 'total_tokens': 24}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_0f03d4f0ee', 'finish_reason': 'stop', 'logprobs': None} id='run-4202b29e-4843-4a65-9de9-4591dc0852d3-0' usage_metadata={'input_tokens': 16, 'output_tokens': 8, 'total_tokens': 24}
```

## Chain과 LCEL(LangChain Expression Language)

```python
from langchain_core.prompts import PromptTemplate

prompt = PromptTemplate.from_template("{country}의 수도는 어디입니까?")
```

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    temperature=0.1,  # 창의성 (0.0 ~ 2.0)
    model_name="gpt-4o-mini",  # 모델명
)
```

```python
from langchain_core.output_parsers import StrOutputParser

output_parser = StrOutputParser()
```

기본적으로 `chain`은 `prompt`, `model`, `parser`의 sequential로 이루어진다. 각 요소는 용도에 따라 다양한 클래스로 구현되어 있으므로 필요에 따라 적절히 가져다 쓸 수 있다.

```python
chain = prompt | llm | output_parser
print(chain)
print(type(chain))
```

```
first=PromptTemplate(input_variables=['country'], template='{country}의 수도는 어디입니까?') middle=[ChatOpenAI(client=<openai.resources.chat.completions.Completions object at 0x10e60bc50>, async_client=<openai.resources.chat.completions.AsyncCompletions object at 0x10e617590>, model_name='gpt-4o-mini', temperature=0.1, openai_api_key=SecretStr('**********'), openai_proxy='')] last=StrOutputParser()
<class 'langchain_core.runnables.base.RunnableSequence'>
```

```python
type(chain)
```

```
langchain_core.runnables.base.RunnableSequence
```

```python
chain.invoke({"country": "대한민국"})
```

```
'대한민국의 수도는 서울입니다.'
```

타입에서 알 수 있듯이 `chain`은 `Runnable`의 sequence이다. `prompt`, `model`, `parser`도 특정한 기능을 수행하고 있는 `Runnable`의 한 종류이다. 그렇기 때문에 아래 코드에서 각 요소들도 개별적으로 `invoke()` method를 호출 가능함을 확인할 수 있다.

```python
print(prompt.invoke({"country": "대한민국"}))
output = llm.invoke("대한민국의 수도는?")
print(output)
print(output_parser.invoke(output))
```

```
text='대한민국의 수도는 어디입니까?'
content='대한민국의 수도는 서울입니다.' response_metadata={'token_usage': {'completion_tokens': 8, 'prompt_tokens': 13, 'total_tokens': 21}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_0f03d4f0ee', 'finish_reason': 'stop', 'logprobs': None} id='run-9c6f9ff5-699c-40ef-ac25-28cc70f33f5a-0' usage_metadata={'input_tokens': 13, 'output_tokens': 8, 'total_tokens': 21}
대한민국의 수도는 서울입니다.
```

## Runnable

chain을 효율적으로 구성하기 위해 연결고리 역할을 하는 `Runnable` 객체들이 있다.

-   `RunnablePassThrough`

-   `RunnableParalle`

-   `RunnableLambda`

### RunnablePassThrough

아래 두 코드는 완전히 동일하다. 즉, `dict`를 chain에 포함시키고자 할 때 `RunnablePassThrough`를 활용할 수 있다.

```python
from langchain_core.runnables import RunnablePassthrough

print(
    prompt.invoke({"country":"대한민국"}) ==
    ({"country": RunnablePassthrough()} | prompt).invoke("대한민국")
)

prompt.invoke({"country":"대한민국"})
```

```
True
```

```
StringPromptValue(text='대한민국의 수도는?')
```

`assign()` method를 이용하여, input으로 받은 `dict`에 새로운 키-밸류 쌍을 추가할 수 있다.

```python
# 입력 키: num, 할당(assign) 키: new_num
(RunnablePassthrough.assign(address=lambda x: x["country"] + " " +  x["city"])).invoke({"country": "대한민국", "city": "서울"})
```

```
{'country': '대한민국', 'city': '서울', 'address': '대한민국 서울'}
```

### RunnableParallel

`RunnableParallel`은 input을 받으면, 이를 지정한 parameter 수 만큼의 `Runnable`(또는 callable, dict)에 전달하여 분기점을 형성하는 객체이다.

```python
from langchain_core.runnables import RunnableParallel

RunnableParallel(out1=lambda x: x, out2=lambda x:x+1, out3=lambda x:x+2).invoke(1)
```

```
{'out1': 1, 'out2': 2, 'out3': 3}
```

`RunnableParallel`을 이용하여 여러 개의 chain을 병렬적으로 연결할 수 있다.

`RunnableParallel`로 chain을 연결할 경우 병렬처리되어 실행시간면에서 이득을 볼 수 있다.

```python
llm = ChatOpenAI(
    temperature=0.1,  # 창의성 (0.0 ~ 2.0)
    model_name="gpt-4o",  # 모델명
)

chain1 = PromptTemplate.from_template("{country}의 수도는 어디입니까?") | llm | StrOutputParser()
chain2 = PromptTemplate.from_template("{country}의 인구는 몇 명입니까?") | llm | StrOutputParser()
chain3 = PromptTemplate.from_template("{country}의 면적은 얼마입니까?") | llm | StrOutputParser()

combined_chain = ({"country": RunnablePassthrough()}
                  | RunnableParallel(capital=chain1, population=chain2, area=chain3))
```

```python
combined_chain.invoke("대한민국")
```

```
{'capital': '대한민국의 수도는 서울입니다.',
 'population': '2023년 기준으로 대한민국의 인구는 약 5,100만 명 정도입니다. 하지만 인구는 지속적으로 변동하므로, 최신 통계는 대한민국 통계청이나 관련 기관의 공식 자료를 참고하는 것이 좋습니다.',
 'area': '대한민국의 면적은 약 100,210 평방킬로미터입니다. 이는 한반도의 남쪽 부분에 해당하며, 북한과 함께 한반도를 구성하고 있습니다.'}
```

### RunnableLambda

`RunnableLambda`를 사용하여 사용자 정의 함수를 맵핑할 수 있다.

입력 변수가 필요하지 않은 함수의 경우

```python
from langchain_core.runnables import RunnableLambda
from datetime import datetime


def get_today(a):
    return datetime.today().strftime("%b-%d")

print(get_today(None))

RunnableLambda(get_today).invoke("")
```

```
Aug-04
```

```
'Aug-04'
```

입력 변수가 1개인 함수의 경우

```python
def get_text_length(text):
    return len(text)

print(get_text_length("pizza"))

RunnableLambda(get_text_length).invoke("pizza")
```

```
5
```

```
5
```

입력 변수가 여러 개인 경우, 인수를 딕셔너리 하나로 받아서 원 함수를 호출하는 형태의 wrapping 함수를 재정의하여 활용해야 한다.

```python
def concat_text(text1, text2):
    return text1 + "-" + text2

def _concat_text(_dict):
    return concat_text(_dict["text1"], _dict["text2"])

print(concat_text("potato", "pizza"))

RunnableLambda(_concat_text).invoke({"text1":"potato", "text2":"pizza"})
```

```
potato-pizza
```

```
'potato-pizza'
```

굳이 함수를 재정의하지 않고 unpacking 연산자와 lambda를 활용하여 간단하게 작성할 수 있다.

```python
def concat_text(text1, text2):
    return text1 + "-" + text2

RunnableLambda(lambda _: concat_text(**_)).invoke({"text1":"potato", "text2":"pizza"})
```

```
'potato-pizza'
```

아래와 같이 활용할 수 있다.

```python
chain = (
    {"today": RunnableLambda(get_today)}
    | PromptTemplate.from_template("{today}에 가까운 기념일을 나열해줘")
    | llm
    | StrOutputParser()
)

display(Markdown(chain.invoke("")))
```

```
<IPython.core.display.Markdown object>
```
