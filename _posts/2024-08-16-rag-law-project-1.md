---
layout: post
title: "LangChain RAG Project - 민법GPT 만들기 "
subtitle: "민법표준판례를 활용한 RAG 실습"
date: 2024-08-16 02:43:00 +0900
comments: True
---

RAG를 적용한 첫번째 프로젝트로, "민법 표준판례연구"를 참고하여 법률적 질문에 대해 답을 줄 수 있는 언어 모델을 구축해보고자 하였다.

ChatGPT는 수학이나 과학, 프로그래밍 분야와는 달리 법률에 관한 질문을 받았을 때 유독 Hallucination이 심각하게 나타난다. 이는 아마도 한국어 학습 자료의 부족함, 단어 하나에 결론이 크게 달라지는 학문적 특수성 등에 기인할 것이다.

Hallucination이 심할수록 RAG의 도입이 강하게 요구된다. 내재된 파라미터 보다 주어진 문서를 바탕으로 답변을 해야 정확한 답을 내릴 수 있고, 어떤 자료를 참고했는지 출처를 명확하게 밝혀야 신뢰성을 담보할 수 있다.

법률, 특히 민법 분야에서 참고할 수 있는 데이터는 생각보다 많지 않다. 보통 종이책으로 전공 서적들이 발간되기 때문이다. 그나마 법전협에서 발간하는 표준판례연구가 민법 전반에 대한 주요 쟁점 판례들을 수록하고 있고, 바로 파일을 구할 수 있어 이를 바탕으로 프로젝트를 진행하였다.

# STEP 1. Document Loading

RAG를 수행하기 위한 정보로 법학전문대학원협의회에서 매년 발간하는 [변호사시험의 자격시험을 위한 민법 표준판례연구](https://info.leet.or.kr/board/board.htm?bbsid=publication&ctg_cd=pds)를 채택하였다. PDF 형식으로 배포되므로, PDF를 불러와서 전처리하는 과정이 필요하다.

```python
%pip install -qU pypdf
```

```output

[1m[[0m[34;49mnotice[0m[1;39;49m][0m[39;49m A new release of pip is available: [0m[31;49m24.1.2[0m[39;49m -> [0m[32;49m24.2[0m
[1m[[0m[34;49mnotice[0m[1;39;49m][0m[39;49m To update, run: [0m[32;49mpip install --upgrade pip[0m
Note: you may need to restart the kernel to use updated packages.
```

```python
from langchain_community.document_loaders import PyPDFLoader

FILE_PATH = "./pdf/민법_표준판례_lite.pdf"
# 파일 경로 설정
loader = PyPDFLoader(FILE_PATH)

# PDF 로더 초기화
docs = loader.load()

# 문서의 내용 출력
print(docs[0].page_content[:100])
print("-"*50)
print(docs[1].page_content[:100])
print("-"*50)
print(docs[2].page_content[:100])
```

```output
제1편  제 1 장  통 칙
3
제1편   총   칙
제1장   통  칙
1.
 민법의 법원：관습법 (1)
(대법원 2003. 7. 24. 선고 2001다48781 전원합의
--------------------------------------------------
변호사시험의 자격시험을 위한 민법표준판례연구
4<판결요지>
  [1] 관습법이란 사회의 거듭된 관행으로 생성한 사회생활규범이 사회의 법적 확신과 인식에 의하
여 법적 규범으로 승인
--------------------------------------------------
제1편  제 1 장  통 칙
5고 함이 상당하다 .
  [6] 대법원이 ‘공동선조와 성과 본을 같이 하는 후손은 성별의 구별 없이 성년이 되면 당연히 그
구성원이 된다.’고
```

빠른 테스트를 위해 첫 10장만 가지고 lite version을 만들어 테스트해본다. 반복적으로 나타나는 "변호사시험의 자격시험을 위한 민법 표준판례연구"와 같은 머릿말이나, 머릿말 다음으로 오는 쪽번호를 제거해줄 필요가 있다.

```python
import re

def preprocess(page_content):
    # 머릿말 없애기
    text = "\n".join(page_content.split("\n")[1:])

    # 쪽번호 없애기
    p = re.compile("\d")
    i=0
    if len(text) == 0: return text
    while p.match(text[i]): i+=1
    text = text[i:]

    # 판례 시작 번호를 기준으로 분리하기
    p = re.compile("\d+[.]\n")
    text = p.sub("\n\n###", text)
    return text

```

```python
for text in map(lambda x:preprocess(x.page_content), docs):
    print(text)
```

깔끔하게 전처리가 된 모습을 확인할 수 있다. 이제 실제 PDF에 적용해보자.

```python
FILE_PATH = "./pdf/민법_표준판례.pdf"
loader = PyPDFLoader(FILE_PATH)
docs = loader.load()
```

```python
content = "\n".join(map(lambda x:preprocess(x.page_content), docs[5:-24]))
len(content)
```

```output
779890
```

머릿말(~5page)과 색인(뒤에서 24page까지)를 제외한 본문 부분을 하나로 연결하여 `content` 변수에 저장하였다. 총 779890 토큰에 해당하는 긴 텍스트 문서가 되었다.

# STEP 2. Text Split

`content` 문서를 적절히 분할하여 RAG로 활용해야한다. 이때, 우리가 가진 표준판례 문서는 판례마다 하나씩 의미를 가지고 있으므로, 판례를 기준으로 분할하는 것이 합리적이다.

```python
%pip install -qU langchain-text-splitters
```

```output

[1m[[0m[34;49mnotice[0m[1;39;49m][0m[39;49m A new release of pip is available: [0m[31;49m24.1.2[0m[39;49m -> [0m[32;49m24.2[0m
[1m[[0m[34;49mnotice[0m[1;39;49m][0m[39;49m To update, run: [0m[32;49mpip install --upgrade pip[0m
Note: you may need to restart the kernel to use updated packages.
```

```python
from langchain_text_splitters import CharacterTextSplitter

text_splitter = CharacterTextSplitter(
    chunk_size=250,
    chunk_overlap=50,
    is_separator_regex=False,
)
```

원래는 `separator`를 지정해서 분리하는게 맞아보이는데, 이상하게 위와 같이 세팅하야 내가 원하는 대로 `\n\n`을 기준으로 텍스트를 분할해주었다.

```python
# text_splitter를 사용하여 state_of_the_union 텍스트를 문서로 분할합니다.
texts = text_splitter.split_text(content)
print(texts[0])  # 분할된 문서 중 첫 번째 문서를 출력합니다.
```

```output
제  1  장   통  칙
제 2 장  인
제  3  장   법  인제  4  장   물  건제 5 장  법률행위제 6 장  소멸시효


제1편   총   칙
제1장   통  칙
```

```python
print(max(list([len(x) for x in texts]))) ## should be less than 8192
print(len(texts))
```

```output
4095
935
```

분할된 chunk의 최대 길이는 4095이고, chunk(판례)의 개수는 총 935개이다.

```python
print(texts[253])
```

```output
### 합유 (1)：합유지분의 승계
(대법원 1996. 12. 10. 선고 96다23238 판결 )
<쟁점>
  합유자 중 일부가 사망한 경우의 소유권 귀속관계
<판결요지>
  [1] 합유로 소유권이전등기가 된 부동산에 관하여 명의신탁해지를 원인으로 한 소유권이전등기절
차의 이행을 구하는 소송은 합유물에 관한 소송으로서 고유필요적 공동소송에 해당하여 합유자 전
원을 피고로 하여야 할 뿐 아니라 합유자 전원에 대하여 합일적으로 확정되어야 하므로, 합유자
중 일부의 청구인낙이나 합유자 중 일부에 대한 소의 취하는 허용되지 않는다 .
  [2] 부동산의 합유자 중 일부가 사망한 경우 합유자 사이에 특별한 약정이 없는 한 사망한 합유
자의 상속인은 합유자로서의 지위를 승계하지 못하므로, 해당 부동산은 잔존 합유자가 2인 이상일
경우에는 잔존 합유자의 합유로 귀속되고 잔존 합유자가 1인인 경우에는 잔존 합유자의 단독소유
로 귀속된다 .
<판례선정이유>
  부동산의 합유자 중 일부가 사망한 경우 상속인이 합유자 지위를 승계하지 못하므로 잔존 합유
자가 2인 이상이면 잔존 합유자의 합유로, 잔존 합유자가 1인이면 그의 단독소유로 각각 귀속함을
밝힌 판결
```

```python
for text in texts[:20]:
    print(text.split("\n")[0])
```

```output
제  1  장   통  칙
### 민법의 법원：관습법 (1)
### 민법의 법원：관습법 (2)
### 민법의 법원： 헌법
### 신의성실의 원칙：사정변경의 원칙 (1)
### 신의성실의 원칙：사정변경의 원칙 (2)
### 신의성실의 원칙：강행법규 위반과의 관계 (1)
###  신의성실의 원칙：강행법규 위반과의 관계 (2)
### 신의성실의 원칙：실효의 원칙
### 신의성실의 원칙： 고지의무
### 신의성실의 원칙： 보호의무
### 신의성실의 원칙：금반언의 원칙
### 신의성실의 원칙： 호의동승
### 권리남용금지의 원칙 (1)
### 권리남용금지의 원칙 (2)
### 권리남용금지의 원칙 (3)
### 태아의 권리능력 (1)
### 태아의 권리능력 (2)
### 의사능력
### 미성년자의 행위능력 (1)
```

Chunk 중 무작위로 하나를 골라 출력해보고, Chunk의 맨 첫줄을 출력해보면 잘 분할되었음을 확인할 수 있다.

# STEP 3. Embedding

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
```

```python
doc_result = embeddings.embed_documents(
    texts
)
```

```python
from sklearn.metrics.pairwise import cosine_similarity

def similarity(a, b):
    return cosine_similarity([a], [b])[0][0]
```

```python
question = '''
채권의 소멸에 관한 설명 중 옳은 것을 모두 고른 것은? (다툼이 있는 경우 판례에 의함)

ㄱ. 임대인은 임대차 존속 중 차임채권의 소멸시효가 완성된 경우 이를 자동채권으로 삼아 임대차보증금 반환채무와 상계할 수 없으나, ｢민법｣ 제495조의 유추적용에 의하여 그 연체차임을 임대차보증금에서 공제할 수는 있다.
ㄴ. 근로자의 경제생활 안정을 해할 염려가 없는 등 특별한 사정이 있어 사용자가 초과 지급된 임금의 부당이득반환청구권으로 근로자의 임금채권과 상계할 수 있는 경우에도, 이러한 사용자의 상계는 임금채권의 2분의 1을 초과하는 부분에 관하여만 허용된다.
ㄷ. 채권양수인이 양수채권을 자동채권으로 하여 채무자가 양수인에 대해 가지고 있던 기존 채권과 상계한 경우, 채권양도 전에 이미 양 채권의 변제기가 도래하였다고 하더라도 상계의 효력은 변제기가 아니라 채권양도의 대항요건이 갖추어진 시점으로 소급한다.
ㄹ. 임대인이 임차인에 대해 갖고 있던 대여금채권의 소멸시효가 임대차 존속 중 완성되었다면 임대인은 위 채권을 자동채권으로 하여 임차인의 임대인에 대한 유익비상환채권과 상계할 수 없다.

'''

embeded_question = embeddings.embed_query(question)
```

```python
import numpy as np

P = np.array(doc_result)
q = np.array(embeded_question)
```

```python
sim = cosine_similarity([q], P).squeeze()
sim.shape
```

```output
(935,)
```

```python
sim[np.argsort(sim)[-5:]]
```

```output
array([0.66670259, 0.66787872, 0.67578467, 0.69705162, 0.71646718])
```

```python
retrieved = [texts[i] for i in np.argsort(sim)[-10:]]
for text in retrieved:
    print(text[:300], end="\n\n\n")
```

# STEP 4. VectorStore

langchain community에서 제공하는 `FAISS` 모듈을 활용하면, chunks로부터 손쉽게 vectorstore를 구현할 수 있다.

```python
from langchain_community.vectorstores import FAISS

db = FAISS.from_texts(texts, embedding=embeddings)
```

```python
query = "문제: 임대인이 임차인에 대해 갖고 있던 대여금채권의 소멸시효가 임대차 존속 중 완성되었다면 임대인은 \
위 채권을 자동채권으로 하여 임차인의 임대인에 대한 유익비상환채권과 상계할 수 없다."
docs = db.similarity_search(query, k=3)
for doc in docs: print(doc.page_content[:150], end="\n\n")
```

```output
###  임차인의 비용상환청구권과 유치권의 성립
(대법원 1975. 4. 22. 선고 73다2010 판결 )
<쟁점>
  임대차종료시에 임차인이 건물을 원상으로 복구하여 임대인에게 명도키로 약정한 경우에 비용상
환청구권이 있음을 전제로 하는 유치권 주장이 가능한지의 여

### 시효완성된 채권을 자동채권으로 하는 상계
(대법원 2021. 2. 10 . 선고 2017다258787 판결
<쟁점>
  임차인이 유익비를 지출한 경우, 임차인의 유익비상환채권의 발생 시기(=임대차계약 종료 시) 및
임대차 존속 중 임대인의 구상금채권 소멸시효가

###  임차권의 무단양도, 무단전대
(대법원 2010. 6. 10. 선고 2009다101275 판결 )
<쟁점>
  임대인의 동의 없이 제3자에게 임차물을 사용·수익하도록 한 임차인의 행위가 임대인에 대한 배
신적 행위라고 할 수 없는 특별한 사정이 있는 경우, 임대

```

vectorstore는 `.as_retriever()` 메소드를 통해 LangChain에서 Chain으로 이어붙일 수 있는 Runnable 객체로 변환할 수 있다.

```python
retriever = db.as_retriever()
```

```python
retriever.invoke("명의신탁과 사해행위취소소송")
```

```output
[Document(page_content='### 사해행위 (3)：신탁자의 신탁재산의 처분행위\n(대법원 2016. 7. 29. 선고 2015다56086 판결 )\n<쟁점>\n  1. 유효인 부부간 명의신탁에서 명의신탁관계가 종료된 경우, 신탁자의 수탁자에 대한 소유권이\n전등기청구권이 신탁자의 책임재산이 되는지 여부\n  2. 신탁자가 유효한 명의신탁약정을 해지함을 전제로 신탁된 부동산을 제3자에게 직접 처분하면\n서 수탁자에게서 곧바로 제3자 앞으로 소유권이전등기를 마쳐 주는 것이 사해행위에 해당하는지 \n여부\n<판결요지>\n  부부간의 명의신탁약정은 특별한 사정이 없는 한 유효하고(부동산 실권리자명의 등기에 관한 법\n률 제8조 참조), 이때 명의신탁자는 명의수탁자에 대하여 신탁해지를 하고 신탁관계의 종료 그것만\n을 이유로 하여 소유 명의의 이전등기절차의 이행을 청구할 수 있음은 물론, 신탁해지를 원인으로 \n하고 소유권에 기해서도 그와 같은 청구를 할 수 있는데, 이와 같이 명의신탁관계가 종료된 경우 \n신탁자의 수탁자에 대한 소유권이전등기청구권은 신탁자의 일반채권자들에게 공동담보로 제공되는 \n책임재산이 된다. 그런데 신탁자가 유효한 명의신탁약정을 해지함을 전제로 신탁된 부동산을 제 3\n자에게 직접 처분하면서 수탁자 및 제3자와의 합의 아래 중간등기를 생략하고 수탁자에게서 곧바\n로 제3자 앞으로 소유권이전등기를 마쳐 준 경우 이로 인하여 신탁자의 책임재산인 수탁자에 대한 \n소유권이전등기청구권이 소멸하게 되므로, 이로써 신탁자의 소극재산이 적극재산을 초과하게 되거\n나 채무초과상태가 더 나빠지게 되고 신탁자도 그러한 사실을 인식하고 있었다면 이러한 신탁자의 \n법률행위는 신탁자의 일반채권자들을 해하는 행위로서 사해행위에 해당한다 .\n<판례선정이유> \n 신탁자가 유효한 명의신탁약정을 해지함을 전제로 신탁된 부동산을 제3자에게 직접 처분하면서 \n수탁자에게서 곧바로 제3자 앞으로 소유권이전등기를 마쳐 주는 것이 사행행위에 해당한다고 본 \n판결'),
 Document(page_content='### 채권자취소권의 행사 (3)：일부의 사해행위와 원상회복 방법\n(대법원 2006. 12. 7. 선고 2006다43620 판결 )\n<쟁점>\n  근저당권설정계약 중 일부만이 사해행위에 해당하는 경우, 원상회복의 방법\n<판결요지>\n  사해행위의 취소에 따른 원상회복은 원칙적으로 그 목적물 자체의 반환에 의하여야 하고, 그것\n이 불가능하거나 현저히 곤란한 경우에 한하여 예외적으로 가액배상에 의하여야 하는바, 근저당권\n설정계약 중 일부만이 사해행위에 해당하는 경우에는 그 원상회복은 근저당권설정등기의 채권최고\n액을 감축하는 근저당권변경등기절차의 이행을 명하는 방법에 의하여야 한다 .\n<판례선정이유> \n  근저당권설정계약 중 일부가 사해행위에 해당하는 경우, 그 원상회복의 방법에 대하여 검토하고 \n있는 판결'),
 Document(page_content='### 사해행위 (4)：상속재산 분할협의\n(대법원 2001. 2. 9. 선고 2000다51797 판결 )\n<쟁점>\n  채무초과 상태에 있는 채무자가 상속재산의 분할협의를 하면서 상속재산에 관한 권리를 포기함\n으로써 일반 채권자에 대한 공동담보가 감소되는 경우, 사해행위 취소의 범위\n<판결요지>\n  [1] 상속재산의 분할협의는 상속이 개시되어 공동상속인 사이에 잠정적 공유가 된 상속재산에 대\n하여 그 전부 또는 일부를 각 상속인의 단독소유로 하거나 새로운 공유관계로 이행시킴으로써 상\n속재산의 귀속을 확정시키는 것으로 그 성질상 재산권을 목적으로 하는 법률행위이므로 사해행위\n취소권 행사의 대상이 될 수 있다 . \n  [2] 채무초과 상태에 있는 채무자가 상속재산의 분할협의를 하면서 상속재산에 관한 권리를 포기\n함으로써 결과적으로 일반 채권자에 대한 공동담보가 감소되었다 하더라도, 그 재산분할결과가 채\n무자의 구체적 상속분에 상당하는 정도에 미달하는 과소한 것이라고 인정되지 않는 한 사해행위로\n서 취소되어야 할 것은 아니고, 구체적 상속분에 상당하는 정도에 미달하는 과소한 경우에도 사해\n행위로서 취소되는 범위는 그 미달하는 부분에 한정하여야 한다 . \n<판례선정이유> \n  상속재산의 분할협의가 사해행위취소권 행사의 대상이 되는지 여부 및 사해행위 취소의 범위를 \n검토하고 있는 판결'),
 Document(page_content='###  불법행위에 대한 금지청구 (2)： 통행방해 \n(대법원 2021. 10. 14. 선고 2021다242154 판결 )\n<쟁점>\n  일반 공중의 통행에 공용된 도로 통행을 방해함으로써 특정인의 통행 자유를 침해한 경우, 민법\n상 불법행위에 해당하는지 여부 및 이때 침해를 받은 자가 통행방해 행위 금지를 소구할 수 있는\n지 여부\n<판결요지>\n  불특정 다수인인 일반 공중의 통행에 공용된 도로, 즉 공로(公路)를 통행하고자 하는 자는 그 도\n로에 관하여 다른 사람이 가지는 권리 등을 침해한다는 등의 특별한 사정이 없는 한, 일상생활상 \n필요한 범위 내에서 다른 사람들과 같은 방법으로 그 도로를 통행할 자유가 있다. 제3자가 특정인\n에 대하여만 그 도로의 통행을 방해함으로써 일상생활에 지장을 받게 하는 등의 방법으로 특정인\n의 통행 자유를 침해하였다면 민법상 불법행위에 해당하고, 침해를 받은 자로서는 방해의 배제나 \n장래에 생길 방해를 예방하기 위하여 통행방해 행위의 금지를 소구할 수 있다 .\n<판례선정이유>\n  특정인의 공로 통행 자유를 침해하면 민법상 불법행위에 해당할 수 있고, 이때 침해를 받은 자\n가 방해배제나 예방을 위하여 통행방해 행위 금지를 소구할 수 있음을 밝힌 판결')]
```

embedding도 API cost를 발생시키기 때문에, 한번 만들어놓고 vector store를 로컬에 저장하여 재사용하는 것이 좋다.

```python
DB_INDEX = "STANDARD_CASE_CIVIL_LAW_INDEX"
db.save_local(DB_INDEX)
```

# STEP 5. Retriever

```python
new_db = FAISS.load_local(DB_INDEX, embeddings, allow_dangerous_deserialization=True)

query = "문제: 임대인이 임차인에 대해 갖고 있던 대여금채권의 소멸시효가 임대차 존속 중 완성되었다면 임대인은 위 채권을 자동채권으로 하여 임차인의 임대인에 대한 유익비상환채권과 상계할 수 없다."
docs = new_db.similarity_search(query, k=3)
for doc in docs: print(doc.page_content[:150], end="\n\n")
```

```output
###  임차인의 비용상환청구권과 유치권의 성립
(대법원 1975. 4. 22. 선고 73다2010 판결 )
<쟁점>
  임대차종료시에 임차인이 건물을 원상으로 복구하여 임대인에게 명도키로 약정한 경우에 비용상
환청구권이 있음을 전제로 하는 유치권 주장이 가능한지의 여

### 시효완성된 채권을 자동채권으로 하는 상계
(대법원 2021. 2. 10 . 선고 2017다258787 판결
<쟁점>
  임차인이 유익비를 지출한 경우, 임차인의 유익비상환채권의 발생 시기(=임대차계약 종료 시) 및
임대차 존속 중 임대인의 구상금채권 소멸시효가

###  임차권의 무단양도, 무단전대
(대법원 2010. 6. 10. 선고 2009다101275 판결 )
<쟁점>
  임대인의 동의 없이 제3자에게 임차물을 사용·수익하도록 한 임차인의 행위가 임대인에 대한 배
신적 행위라고 할 수 없는 특별한 사정이 있는 경우, 임대

```

# RAG Chain 구성

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import PyMuPDFLoader
from langchain_community.vectorstores import FAISS
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from IPython.display import display, Markdown

# Load vectorstore and convert to retriever
DB_INDEX = "STANDARD_CASE_CIVIL_LAW_INDEX"
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
db = FAISS.load_local(DB_INDEX, embeddings, allow_dangerous_deserialization=True)
retriever = db.as_retriever(search_kwargs={"k": 2})

# prompt setting
prompt = PromptTemplate.from_template(
    """당신은 민법 전문가입니다. 다음 Context를 참고하여 Qustion에 대한 답변을 하고, Context를 참고한 경우 참조한 판례번호를 모두 병기해주세요.

#Question:
{question}
#Context:
{context}

#Answer:"""
)


llm = ChatOpenAI(model_name="gpt-4o", temperature=0)

chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
```

# Chain 실행

```python
question = "사해행위 취소로 인한 원상회복으로 부동산을 반환하는 경우 그 사용이익의 반환여부?"
print("# 질문 \n" + question)
for i, doc in enumerate(retriever.invoke(question)):
    print(f"\n\n# 참고 판례 {(i+1)}")
    print((doc.page_content))
print("\n\n# 답변")
print(chain.invoke(question))
```

```output
# 질문
사해행위 취소로 인한 원상회복으로 부동산을 반환하는 경우 그 사용이익의 반환여부?


# 참고 판례 (1)
### 채권자취소권의 행사 (5)：원상회복 시 사용이익의 반환 여부
(대법원 2008. 12. 11. 선고 2007다69162 판결 )
<쟁점>
  사해행위 취소로 인한 원상회복으로 부동산을 반환하는 경우에 그 사용이익이나 임료상당액도
반환해야 하는지 여부
<판결요지>
  채권자취소권은 채무자가 채권자를 해함을 알면서 일반재산을 감소시키는 행위를 한 경우에 그
행위를 취소하여 채무자의 재산을 원상회복시킴으로써 채무자의 책임재산을 보전하기 위하여 인정
된 권리로서, 사해행위의 취소 및 원상회복은 책임재산의 보전을 위하여 필요한 범위 내로 한정되
어야 하므로 원래의 책임재산을 초과하는 부분까지 원상회복의 범위에 포함된다고 볼 수 없다. 따
라서 부동산에 관한 법률행위가 사해행 위에 해당하여 제406조 제1항에 의하여 취소된 경우에 수
익자 또는 전득자가 사해행위 이후 그 부동산을 직접 사용하거나 제3자에게 임대하였다고 하더라
도, 당초 채권자의 공동담보를 이루는 채무자의 책임재산은 당해 부동산이었을 뿐 수익자 또는 전
득자가 그 부동산을 사용함으로써 얻은 사용이익이나 임차인으로부터 받은 임료상당액까지 채무자
의 책임재산이었다고 볼 수 없으므로 수익자 등이 원상회복으로서 당해 부동산을 반환하는 이외에
그 사용이익이나 임료상당액을 반환해야 하는 것은 아니다 .
<판례선정이유>
  사해행위 취소로 인한 원상회복으로 부동산을 반환하는 경우에 그 사용이익이나 임료상당액도
반환해야 하는지 여부를 검토한 판결


# 참고 판례 (2)
### 대상청구권
(대법원 2012. 6. 28. 선고 2010다71431 판결
<쟁점>
  사해행위취소소송에서 원물반환으로 근저당권설정등기의 말소를 구하여 승소판결이 확정되었는
데, 부동산이 담보권 실행을 위한 경매절차에 의하여 매각됨으로써 확정판결에 기한 수익자의 근저
당권설정등기 말소등기절차의무가 이행불능된 경우에 대상청구권이 인정되는지 여부
<판결요지>
  [1] 우리 민법이 이행불능의 효과로서 채권자의 전보배상청구권과 계약해제권 외에 별도로 대상
청구권을 규정하고 있지 않으나 해석상 대상청구권을 부정할 이유는 없다 .
  [2] 신용보증기금이 갑 주식회사를 상대로 제기한 사해행위취소소송에서 원물반환으로 근저당권
설정등기의 말소를 구하여 승소판결이 확정되었는데, 그 후 해당 부동산이 관련 경매사건에서 담보
권 실행을 위한 경매절차를 통하여 제3자에게 매각된 사안에서, 위와 같이 부동산이 담보권 실행
을 위한 경매절차에 의하여 매각됨으로써 확정판결에 기한 갑 회사의 근저당권설정등기 말소등기
절차의무가 이행불능된 경우, 신용보증기금은 대상청구권 행사로서 갑 회사가 말소될 근저당권설정
등기에 기한 근저당권자로서 지급받은 배당금의 반환을 청구할 수 있다고 한 사례
<판례선정이유>
 채권자취소사해행위취소소송에서의 피고인 수익자의 원물반환채무의 이행이 불능이 된 경우에도
대상청구권을 인정할 수 있다고 설시한 판결


# 답변
사해행위 취소로 인한 원상회복으로 부동산을 반환하는 경우, 그 사용이익이나 임료상당액을 반환해야 하는지 여부에 대해 대법원은 다음과 같이 판시하였습니다.

채권자취소권은 채무자가 채권자를 해함을 알면서 일반재산을 감소시키는 행위를 한 경우에 그 행위를 취소하여 채무자의 재산을 원상회복시킴으로써 채무자의 책임재산을 보전하기 위하여 인정된 권리입니다. 따라서 사해행위의 취소 및 원상회복은 책임재산의 보전을 위하여 필요한 범위 내로 한정되어야 하며, 원래의 책임재산을 초과하는 부분까지 원상회복의 범위에 포함된다고 볼 수 없습니다.

따라서 부동산에 관한 법률행위가 사해행위에 해당하여 민법 제406조 제1항에 의하여 취소된 경우에 수익자 또는 전득자가 사해행위 이후 그 부동산을 직접 사용하거나 제3자에게 임대하였다고 하더라도, 당초 채권자의 공동담보를 이루는 채무자의 책임재산은 당해 부동산이었을 뿐, 수익자 또는 전득자가 그 부동산을 사용함으로써 얻은 사용이익이나 임차인으로부터 받은 임료상당액까지 채무자의 책임재산이었다고 볼 수 없으므로, 수익자 등이 원상회복으로서 당해 부동산을 반환하는 이외에 그 사용이익이나 임료상당액을 반환해야 하는 것은 아닙니다.

참조한 판례번호: 대법원 2008. 12. 11. 선고 2007다69162 판결
```
