
# 목차

1. [소개](#소개)
   1. [예상독자](#예상독자)
      1. [예상독자](#예상독자)
      2. [예상독자가 아닌 분](#예상-독자가-아닌-분-)

2. [검색(Fulltext-Search) 알아보기](#검색fulltext-search-알아보기)
   1. [Fulltext search](#fulltext-search-)
      1. [문서가 유사한지 판단하는 기준](#문서가-유사한지-판단하는-기준)
   2. [문서 유사도 알고리즘](#문서-유사도-알고리즘)
      1. [token](#token)
      2. [대표적 문서 유사도 알고리즘 : TF-IDF](#대표적-문서-유사도-알고리즘--tf-idf)
      3. 3. [bm25](#bm25)

3. [Inverted Index](#inverted-index)
    1. [가장 기본적인 개념의 Invereted Index](#가장-기본적인-개념의-invereted-index-)
    2. [value 가 포함된 Inverted Index](#value-가-포함된-inverted-index-)
    3. [Trie 를 이용해 저장 공간을 절약한 Inveted index (elasticsearch)](#trie-를-이용해-저장-공간을-절약한-inveted-index-elasticsearch)
    4. [그럼 쓰기는 나쁜가요?](#그럼-쓰기는-나쁜가요)


4. [analyzer](#analyzer-)
    1. [analyzer란?](#analyzing-란-)
    2. [형태소 분석 analzyer](#형태소-분석-analzyer)
    3. [n-gram](#n-gram-)
    

5. [doc_value와 column based 자료형](#doc_value-와--column-based-자료형)
    1. [column based 자료형](#column-based-자료형-)
    2. [doc_value 사용예시](#doc_value-사용예시-)


---


# 소개

이 문서는 검색 서비스에 입문하시는 분들을 위한 배경지식 모음입니다. 

검색은 검색에서만 사용되는 고유한 컨셉이 진입 장벽이 되기 때문에 이런 배경지식들을 모아서 볼 수 있으면 좋겠다는 컨셉에서 작성된 문서입니다.

검색에 관련한 문서를 읽기 위해 필요한 배경지식들이 있으며, elasticsearch 에서는 implementation 이 어떻게 되어 있는지에 대한 설명이 같이 작성이 되어 있으니 목차를 보고 필요한 부분만 읽으셔도 무방합니다.


이러한 배경 설명없이 바로 튜토리얼로 넘어가고 싶은 분들을 위해 8버전 기준으로 작성한 튜토리얼을 참고하시면 됩니다. 
- 링크 : [es 와 키바나 8버전 https 보안 설정 implmentation](#https://github.com/dkGithup2022/es8_kibana_https_with_self_signed_ca)
- 링크 : [es 쿼리작성 best practice]()


**벡터 검색**에 관해서는 내용이 많아 별도의 문서로 만들었으니, 그 벡터검색의 컨셉이 궁금하다면 아래 링크를 참고해 주시기 바랍니다.
- 링크 : [벡터검색의 컨셉과 고전적인 문서유사도 알고리즘과 비교](#https://github.com/dkGithup2022/es_vector_search_and_comparison_of_bm25_algorithm_search)

---

## 예상 독자

### 예상독자

1. 엘라스틱 서치나 검색에 입문하기 전에 개념적인 내용에 대해 확인하고 싶은 사람
2. ES 관련 문서를 읽을때, TF-IDF, BM25, Token, doc_value ... 등의 용어가 어렵게 느껴지는 사람
3. 그냥 한번 볼 사람 ( 착한사람 ) 

### 예상 독자가 아닌 분 

1. 벡터 검색을 보러온 독자 
   - 벡터검색은 내용이 너무 많아서 따로 문서로 만들었어요. 링크 : [벡터검색의 컨셉과 고전적인 문서유사도와 비교](#https://github.com/dkGithup2022/es_vector_search_and_comparison_of_bm25_algorithm_search)

2. ES 의 API 호출 방법과 각 파라미터의 사용법을 알고 싶은 독자
    - 공식문서가 더 잘 설명해 줄 것입니다. 이 문서는 사용법보단 원론적인 내용에 가깝습니다.
    - 공식문서 링크 : https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html


3. 튜닝이나 원리보다는 지금 당장 빠른 시간 안에 프로덕션에 적용해서 기능을 만들어야 하는 독자
    - 한국어 문서 중 빠르게 적용하기 좋은 문서가 있습니다 : https://esbook.kimjmin.net/


4. 설치랑 https config에서 막한 사람  
   - 이거 여기서 막힌 사람이 많아서 (부끄러워 하지 마시오) 이전에 작성한 가이드를 첨부합니다.
   - 링크 : [es 와 키바나 8버전 https 보안 설정 implmentation](#https://github.com/dkGithup2022/es8_kibana_https_with_self_signed_ca)
   
<br/>

---

<br/>

# 검색(Fulltext-Search) 알아보기

검색 서비스를 만드려면 검색어와 각 문서가 얼마나 **유사한지를 숫자로 표현하는 과정**이 필요합니다.

이 챕터에서는 문서간의 유사한 정도를 숫자로 바꾸는데 필요한 과정과 컨셉에 대해서 설명합니다.

<br/>

## Fulltext search 

검색어에 대해 문서집합에서 **유사한 문서**를 찾아주는 기술/서비스를 Fulltext Search라고 합니다..

문서를 유사한 정도로 줄 세우려면 문서가 유사한 정도를 숫자로 변환할 수 있어야 합니다. 
이런 유사한 정도를 숫자로 바꾸는 방법을 **문서 유사도 알고리즘**이라고 합니다. 

즉, Fulltext search는 개발자가 정한 문서유사도 알고리즘에 따라서 문서집합에서 문서를 유사도 점수가 높은 순서대로 유저에게 서빙해주는 방식이라고 볼 수 있습니다.

### 문서가 유사한지 판단하는 기준

문서가 얼마나 유사한지에 대한 정의는 시대에 따라서 조금씩 다를수도 있지만 크게 두 가지 관점으로 나뉩니다.

1. 문서를 토큰으로 나눈 결과를 **문서 유사도 알고리즘의 결과**로 순서대로 줄 세운 것.
2. 문서를 sentence transformer 모델로 변환한 뒤 **벡터간 거리가 가까운 순**으로 줄 세우는 것.

- [문서 유사도 알고리즘이란](#문서-유사도)

고전적인 의미에서의 검색은  **문서유사도**만 말하는 것이였으나 2021년 들어서부터는  벡터 검색에 대한 관심이 크게 커지고 있습니다.

벡터 검색을 간단하게 설명하면 문서의 string 본문을 vector 로 치환을 한 다음, 이 벡터간의 유사한 정도를 나열하는 방식입니다.

텍스트를 직접 조작할때는 잡을 수 없는 케이스가 벡터상에서는 가까운 위치에 있기 때문에 쉽게 양질의 검색 서비스를 만들 수 있습니다.
가령, "데이터베이스"와 "database" 는 문자의 비교로 봤을 땐, 유사한 부분이 없지만 벡터상에서는 가까운 위치에 위치하기 때문에 "database" 로 검색을 해도 "데이터베이스" 가 포함된 문서를 가져올 수 있습니다.

벡터검색 이용사례와 사용후기에 대해 자세히 보고 싶다면 아래 링크를 참고해주세요

[벡터검색 바로가기](https://github.com/dkGithup2022/es_vector_search_and_comparison_of_bm25_algorithm_search)

## 문서 유사도 알고리즘
**문서끼리 유사한 정도를 숫자로 바꾸는 알고리즘**을  문서 유사도 알고리즘라고 한다.

숫자로 바꾼다는 컨셉만 일치한다면 개발자가 어떤 방식을 선택/창조할지는 본인의 선택입니다. 가령, 아래의 간단한 방식도 일종의 문서 유사도 알고리즘이라고 볼 수 있습니다.


- **검색어**
```
검색어 : 바나나 
````


- **문서집한**
```
문서 A = "바나나 바나나"
문서 B = "딸기 바나나" 
```

- **문서유사도 알고리즘 : 겹치는 횟수를 세는 경우**
```
문서 A 의 단어가 겹치는 횟수: 2 
문서 B 의 단어가 겹치는 횟수: 1
```


가령, 겹치는 단어가 있을 때마다 +1 로 계산하는 문서 유사도 알고리즘이 있다고 치면 위의 환경에서 문서를 찾을 때 "바나나" 라는 검색어에 대해
A 는 2점, B 는 1점으로 A 를 반환할 수 있습니다.

실제로 자주 사용되는 방법은 저것보다 복잡합니다. [전체 문서집합의 단어(토큰)의 빈도를 고려하여 자주 사용되는 단어는 가중치를 낮추는 방법](#tf-idf)이 몇십년전부터 쓰여왔던 메타이지요.  
이 기조의 방법들을 TF-IDF 기반의 방식이라고 합니다. 


<br/>



### Token

**어떤 문서가 하나의 string 이라면 string 덩어리끼리는 비교가 불가능하기 때문에 비교할 수 있는 형태로 바꾸는 전처리가 필요합니다.** 
보통, 문서를 토큰의 배열로 쪼게서 봅니다. 이 토큰이 얼마나 일치하는지가 문서 유사도를 파악의 기초입니다. 


그리고 토큰을 만드는 규칙을 어떻게 정하느냐에 따라서, 검색의 퀄리티가 결정됩니다.
아래에서 두가지 간단한 예시를 볼 것입니다 


`1.  띄어쓰기 기준으로만 토큰을 나눔`

`2.  글의 의미 단위로 토큰을 나눔.`

같은 글을 두가지 기준에 따라 token 배열로 만들고 검색 결과를 비교한 예시를 볼 것입니다. 검색에 어느 정도의 유연함을 추가할지는 각자의 환경과 서비스에 맞게 판단하시면 됩니다.

<br/>


- **띄어쓰기 기준의 token으로 문서를 나눈 예시**


| 문서 텍스트                    | 형태소 분석 후 토큰                          | "와우" 검색어에 매칭 여부 |
|--------------------------------|-------------------------------------------|-----------------|
| "와우 요즘에 재미있나요?"          | ["와우", "요즘에", "재미있나요?"]                    | O               |
| "와우가 요즘 망한거 같다"          | ["와우가", "요즘", "망한거", "같다"]                   | X               |
| "와우는 옛날에 비하면 괜찮어"      | ["와우는", "옛날에", "비하면", "괜찮어"]         | X               |

"와우가","와우는" 이라는 토큰은 "와우"라는 토큰과는 일치하지 않습니다. 따라서 직관적으로 동작하는 검색을 만드려면 형태소분석기를 통해 문서를 의미 단위의 토큰으로 만드는 과정이 필요합니다. 

<br/>


- **형태소 분석기를 통해, 의미 단위의 token 으로 나눈 문서 예시** 

| 문서 텍스트                    | 형태소 분석 후 토큰                          | "와우" 검색어에 매칭 여부 |
|--------------------------------|-------------------------------------------|-----------------|
| "와우 요즘에 재미있나요?"          | ["와우", "요즘", "재미"]                    | O               |
| "와우가 요즘 망한거 같다"          | ["와우", "요즘", "망하-"]                   | O               |
| "와우는 옛날에 비하면 괜찮어"      | ["와우", "옛날", "비하-", "괜차하-"]         | O               |

<br/>

 
위의 예시를 보면 단순히 띄어쓰기로 토큰을 나누는 방식으로는 유저가 원하는 정보를 찾아갈 수 없다는 것을 확인할 수 있습니다.


만약 검색 서비스를 테스트를 해 보았는데 검색어에 대한 검색의 너무  넓거나 좁다면 tokenizing 규칙에 문제가 있는지 살펴보아야 합니다.



<br/>

### 대표적 문서 유사도 알고리즘 : TF-IDF

위의 [문서 유사도 알고리즘](#문서-유사도-알고리즘) 언급한 토큰의 빈도를 고려한 문서유사도 알고리즘 방법을 TF-IDF 라고합니다. TF-IDF 는 전체 문서에서 **언급된 횟수가 많을수록 해당 토큰의 가중치를 줄이는 방법입니다.**


아래에서 같은 문서집합과 검색어에서 가중치 측정을 다르게 함으로서 유저에게 노출되는 검색의 결과가 달라지는 예시를 작성해 놓았습니다.

<br/>

##### 비교용 데이터

- **문서 집합**
```
문서 A : "쿠버네티스란  이러 저러 하다"
문서 B : "이거는 내가 어제 산거고, 이거는 우리집에 원래 있던거야 "
문서 C : "이것(이거)은 큰 발견입니다."
```

- **검색어**

`검색어 : "쿠버네티스 이거 좋은건가요?"`


<br/>

##### 토큰 별 가중치를 고려하지 않은 경우 

정상적으로 토큰을 나눈 이후, 모든 토큰의 가중치가 같다면 의미 없는 내용의 매칭이 많아질 수 있다는 예시입니다.

즉, 검색이 합리적이고 직관적으로 작동하지 않게 된다.


<br/>

 - **검색어 토큰**

| 검색어 | 검색어 토큰                |
|-----|-----------------------|
|  "쿠버네티스 이거 좋은건가요?"   | ["쿠버네티스", "이거", "좋-"] |

- **문서집합 토큰**

| 문서 텍스트                    | 형태소 분석 후 토큰                                                  | ["쿠버네티스","이거","좋-"]  검색어 토큰과 일치 갯수 |
|--------------------------------|--------------------------------------------------------------|------------------------------------|
| "쿠버네티스란 이러 저러 하다"     | [**"쿠버네티스"**, "이러", "저러", "하-"]                              | 쿠버네티스: 1점                          |
| "이거는 내가 어제 산거고, 이거는 우리집에 원래 있던거야" | [**"이거"**, "내", "어제", "사-", **"이거"**, "우리", "집", "원래", "있-"] | 이거: 2점                             |
| "이것(이거)은 큰 발견입니다."   | ["이것", **"이거"**, "큰", "발견하-"]                                | 이거: 1점                             |



<br/>

모든 토큰의 가중치가 같다면 B가 1위입니다. "쿠버네티스" 에 대해 검색 했는데, "이거" 라는 짜투리 단어가 많이 겹쳐서 스코어를 올려버린 것을 볼 수 있습니다.



선배 엔지니어들은 이 단계에서 **"is", "a", "이", "그" 같은 많은 문서에서 언급되는 토큰의 가중치를 의도적으로 줄이는** 선택지를 선택했다.

이 빈도를 고려한 문서 유사도 알고리즘을  TF-IDF 라고 한다.

<br/>

##### TF-IDF 계산식

TF-IDF 는 TF 밸류와 IDF 밸류의 곱이다. TF, IDF 가  각각 의미하는 바는 다음과 같습니다.

TF란 해당 document 내부에서 언급된 횟수, IDF 란 전체 document 에서 언급된 비율로 나누는 것을 말합니다. tf-idf 밸류는 tf 값과 idf 값을 곱한 것을 말합니다.

즉, tf의 의미는 어떤 단어(토큰)에 대해 도큐먼트에서 언급된 횟수가 높을수록 스코어가 올라가고, idf 의 의미는 전체 도큐먼트에서 언급된 횟수가 많을수록 떨어진다는 뜻입니다.

식으로 나타내면 아래와 같습니다.

```
TF =  {document 내 해당 토큰이 언급된 수} / {document 내 전체 토큰 수}
IDF = Log( {전체 문서 수} / ( 1+ {토큰을 포함하는 문서 수}))
TF-IDF = 각 토큰의 TF*IDF 를 누계한 것

```


| 문서 텍스트                    | 형태소 분석 후 토큰                                                  | ["쿠버네티스","이거","좋-"]  검색어 토큰과 일치 갯수     |
|--------------------------------|--------------------------------------------------------------|----------------------------------------|
| "쿠버네티스란 이러 저러 하다"     | [**"쿠버네티스"**, "이러", "저러", "하-"]                              | 쿠버네티스:  1/4 * log(3/ (1+1))   =  0.04  |
| "이거는 내가 어제 산거고, 이거는 우리집에 원래 있던거야" | [**"이거"**, "내", "어제", "사-", **"이거"**, "우리", "집", "원래", "있-"] | 2/9 * log(3/(1+2))     = 0             |
| "이것(이거)은 큰 발견입니다."   | ["이것", **"이거"**, "큰", "발견하-"]                                | 이거: 1점      1/4 * log(3/(1+2))    =  0 |

위의 경우에서는 idf 계산 식에 의해 "이거" 의 스코어를 0으로 만들어버린 것을 확인 할 수 있다. 
전체 문서 집합이 4개인 경우의 B 의  tf-idf 스코어는 2/9*(log(4/3)) = 0.028로 여전히 문서 A가 상단에 있음을 확인할 수 있다. 


위의 TF 식과 ,IDF 식은 고정적인 것이 아니며 개발자의 선택에 의해 커스텀할 수 있다.

전체 문서에서의 토큰 비중, 각 문서내에서의 토큰 비중을 고루 비교하여 스코어를 매기는 방법을 뭉뚱그려서 TF-IDF 라고 말하기도 한다 .
그래서 아래에 설명할 BM25 알고리즘은 BM25 라고도 부르지만 TF-IDF 라도 부르기도 한다. 만약 BM25 를 보고 TF-IDF 라고 말하는 글을 본다면 당황하지 말고, "아 같은 계열이라는 뜻이구나"라고 생각하면 된다.

<br/>


### BM25

TF-IDF 방법의 일종으로, Elasticsearch 의 default 문서유사도 알고리즘입니다. 일반적인 TF-IDF 와의 차이점은 아래와 같다.


1. 특정 토큰의 가중치를 선형적으로 증가시키지 않고, 그보다 느리게 증가시킵니다.
2. TF 계산 시 커스터마이징 가능한 파라미터를 추가하여, 가중치 증가 속도를 조절할 수 있습니다.

```
BM25 = IDF * (TF * (k+1)) / (TF+K*(1-b+b*(LENG_VAL)))
LENG_VAL = {해당 doc 길이} / {전체 doc 평균 길이}


- k : 스무딩 파라미터로, 값이 클수록 단어 빈도의 영향이 커집니다. 일반적으로 0에서 2 사이의 값을 사용합니다.
- b : 길이 보정 파라미터로, 문서 길이에 따른 보정을 조절합니다. b가 0이면 문서 길이가 TF 스코어에 영향을 미치지 않고, b가 1이면 문서 길이에 따른 감점이 있습니다. 일반적으로 0에서 1 사이의 값을
사용합니다. 
```

elasticsearch에서는 저 b와 k 값을 파라미터로서 커스터마이징할 수 있습니다만 추천하지는 않습니다.


검색의 세계에서는 "인기 지표의 반영", "단어 사전의 최신화", "유저에 반응에 대한 ab 테스트" 등에 비하면 bm25 의 가중치 튜닝은 상대적으로 사소한 부분이라고 생각됩니다.

아래에 더 자세한 링크를 남기긴 하겠으나 bm25 에 대한 스코어링 튜닝은 상대적으로 뒤로 미뤄도 되는 부분이라 생각합니다.

[1. 공식 elastic blog: bm25 에 대하여 더 자세한 설명](https://www.elastic.co/kr/blog/practical-bm25-part-2-the-bm25-algorithm-and-its-variables)

[2. 8.14 버전에서 bm25 score param 튜닝하기](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html)


<br/>


## Inverted Index

Inverted 는 주로 검색에 쓰이는 자료형입니다. 

이 챕터에서는 기본적인 inverted index 개념과 elasticsearch 에서는 구현이 어떻게 되어 있는지에 대해 다룹니다.


- **목적**

아래 목적을 빠르게 파악하는 것이 이 자료형의 목적입니다.

1. 검색어와 일치하는 토큰이 문서집합에 있는지
2. 그 문서의 위치(pointer) 가 어디에 있는지


- **시간 복잡도**

Inverted 인덱스의 조회 시간 복잡도는 O(1)입니다.

가끔, O(size of { 검색어 }) 로 표현하는 경우도 있는데, O(1) 로 봐도 무방합니다. 


### 가장 기본적인 개념의 Invereted Index 

![simple_inverted_index.png](images%2Fsimple_inverted_index.png)

inverted 인덱스의 컨셉을 간단하게 나타낸 것입니다.

토큰에 해당하는 도큐먼트 위치를 빠르게 파악할 수 있는 방법은 map의 key 에 대한 get 요청만 있으면 됩니다. 
따라서 이 단계에서 논리적인 시간 복잡도는 O(1) 이라고 볼 수 있습니다.

그래서 데이터를 저장할 때, 토큰을 key(index) 로 가지고 도큐먼트의 포인터를 value 로 가지는 자료형으로 데이터를 저장하는 방식을 inverted index 라고 합니다.

보통, 일반적인 수준에서의 inverted index 를 말하면 이 개념을 말합니다.


`위에서 " 논리적인 시간 복잡도"라는 표현을 썻는데, 실제 검색에서는 성능에서 물리 제약이 더 클 수 있습니다. 해당 내용은 아래의 Trie 로서의 inverted index 와 lucene internal architecture 링크를 참고해주세요`

- 링크 :[es_internal_arcchitecture_and_lucene](https://github.com/dkGithup2022/es_internal_arcchitecture_and_lucene)

### value 가 포함된 Inverted Index 



그런데 위의 방식도 몇가지 문제가 있는데, 실제 token 의 포함여부도 중요하지만, tf-idf 를 계산하려면 빈도도 같이 저장되어야 합니다. 

따라서 실제로 inverted index 를 저장할 땐, 각 토큰의 빈도수와 토큰의 문서 내 위치를 같이 저장한 형태를 사용하게 됩니다.  

- Inverted index 의 예시 : https://www.baeldung.com/cs/indexing-inverted-index
![inverted_index_with_count.png](images%2Finverted_index_with_count.png)

실제 implemenataion 을 반영하려고 ,terms 와 index 를 나누어 표현해놨는데 terms index 테이블을 아래처럼 합쳐서 봐도 무방합니다.

| DocId | term  | doc index  => {docId} : {빈도} : {단어 위치}|
|-------|-------|-----------------------------------------------|
| 1     | be    | 1:2:[2:6]<br/>2:1:[2]<br/>3:1:[3]             |
| 2     | left  | 3:1:[3]                                       |
| 3     | not   | 1:1:[4]<br/>3:1:[1]                           |
| 4     | or    | 1:1:[3]                                       |
| 5     | right | 2:1:[3]                                       |
| 6     | to    | 1:2:[1,5]<br/>2:1:[1]<br/>3:1:[2]             |

의 예시를 보면, Index 영역의 데이터가 ***{docId} : {빈도} : {단어 위치}*** 형태로 저장이 된다.
가령, 첫 줄은,

`"be" 라는 토큰에 대해서 1번 문서는 be 토큰이 언급되고, 위치는 2위치, 6위치에 해당 토큰이 있다.` 

로 해석할 수 있어요.

각 문서 내의 빈도를 같이 저장함으로서 TF-IDF 등의 계산을 더 쉽게 할 수 있습니다.


이 단어의 빈도는 TF-IDF , BM25 등의 문서 유사도 알고리즘을 적용 할 수 있게 하는 파라미터가 됩니다.



### Trie 를 이용해 저장 공간을 절약한 Inveted index (elasticsearch)



수억건의 데이터에 대해 inverted index 를 구성하면 inverted index 키 값의 갯수는 수백억개가 될 수도 있습니다.

inverted index 가 논리적으로 빠르다고 한들, 물리적으로 이런 양의 데이터는 애초에 ram 위에 올라가는 것을 보장할 수 없습니다.
disk io 의 비용이 inverted index 의 장점을 상쇄해버리기도 애초에 invereted index 사전 데이터 자체가 ram 보다 커질 가능성도 있는 구조이기 때문입니다. 

엘라스틱서치 (루씬)에서는 실제로는 inverted index 의 테이블을 trie 압축 구조로 저장하여 저장합니다.

데이터를 압축해서 저장하기 때문에 수억건의 데이터를 가진  inverted index 에 대해서도 상대적으로 빠른 검색을 수행할 수 있습니다.

- 이미지 출처 geeks for geeks
![trie.png](images%2Ftrie.png)

더 자세한 내용은 따로 작성한 [es_internal_arcchitecture_and_lucene](https://github.com/dkGithup2022/es_internal_arcchitecture_and_lucene)
를. 참고해 주세요


### 그럼 쓰기는 나쁜가요?


```
"어? 그러면 읽기는 빠르다는걸 알겠는데, "쓰기" 하면 전체 토큰 집합이 뒤집어질 수도 있는데, 진짜 빠른거 맞음 ? 
쓰기 연산에서 lock이라도 걸리면 뭐 되는게 없을수도 있는거 같은데, 이거 사기 아님?"
```

라는 의문을 가진 똑똑한 독자가 있을 수 있는데 이 궁금함을 해결하려면 따로 작성한 [es_internal_arcchitecture_and_lucene](https://github.com/dkGithup2022/es_internal_arcchitecture_and_lucene) 를 참고해주시길 바랍니다.

결론을 말하자면, 해당 구조는 쓰기에 취약한 구조가 맞다. 그래서 쓰기는 몰아서 일괄적으로 처리하고 쓰기에 대한 transaction 을 보장하지 않는 방식으로 루씬과 Elasticsearch 는 이런 단점을 회피했다.

어떻게 보면 높은 fulltext search 조회 성능을 가지기 위한 일반적이지 않은 기교를 부렸기 때문에,  transaction이 보장이 안된다는 점은 elasticsearch 가 필연적으로 가지게 된 약점이기도 하다. 

---

##  analyzer 

### analyzing 란 ?

analyzing 이란 텍스트를 검색을 위해 전처리 하는 것을 말한다. 크게 두가지 과정으로 나뉘는데 tokenizing 과 filter 이다.

- Tokenizing : 텍스트를 토큰으로 쪼게는 것
- Filter : 텍스트를 규칙에 맞게 바꾸는 것 (lowercase, html 제거 등)

tokenizing 이라는 개념을 analyzing 이 포함하고 있으나, 하지만 여러 세미나나 아티클에서 두 단어를 혼용해서 쓰는 경우도 적지는 않다.


### 형태소 분석 analzyer

위의 예시로 든 경우입니다. 만약에 검색어가 입력 되었을 때, 기계적인 일치여부를 판단할게 아니라 어느정도 문맥에 맞는지를 검사하고 싶다면 형태소 분석 analyzer 를 analyzer 에 적용하시면 됩니다.

언어별로 analzyer 의 종류가 다를 수 있는데, 영어의 경우는 es 8 을 설치하면 내장이 되어 있고 한글의 경우 플러그인을 설치해야 한다. 

형태소 단위로 나누는 경우의 이해를 위해 두가지 예시를 첨부합니다.

1. 형태소를 나눈 후, 모든 형태소를 token의 배열로 만든 경우
2. nori - plug in 의 stop word 옵션을 통해, "은","는","이","가" 등의 의미없는 조사를 제거한 경우

두가지 모두 예시를 첨부했으니 각자 필요하다고 생각하는 만큼 잘라 가시면 된다. 

- nori tokenizer 적용 예시
```json
GET _analyze
{
  "tokenizer": "nori_tokenizer",
  "text": [
    "토큰이란 문서 유사도 알고리즘에서 유사성을 판단하는 기준이다."
  ]
}
```

- nori tokenizer 실행예시 
```json
{
  "tokens": [
    {
      "token": "토큰",
      "start_offset": 0,
      "end_offset": 2,
      "type": "word",
      "position": 0
    },
    {
      "token": "이",
      "start_offset": 2,
      "end_offset": 3,
      "type": "word",
      "position": 1
    },
    {
      "token": "란",
      "start_offset": 3,
      "end_offset": 4,
      "type": "word",
      "position": 2
    },
    {
      "token": "문서",
      "start_offset": 5,
      "end_offset": 7,
      "type": "word",
      "position": 3
    },
    {
      "token": "유사",
      "start_offset": 8,
      "end_offset": 10,
      "type": "word",
      "position": 4
    },
    {
      "token": "도",
      "start_offset": 10,
      "end_offset": 11,
      "type": "word",
      "position": 5
    },
    {
      "token": "알고리즘",
      "start_offset": 12,
      "end_offset": 16,
      "type": "word",
      "position": 6
    },
    {
      "token": "에서",
      "start_offset": 16,
      "end_offset": 18,
      "type": "word",
      "position": 7
    },
    {
      "token": "유사",
      "start_offset": 19,
      "end_offset": 21,
      "type": "word",
      "position": 8
    },
    {
      "token": "성",
      "start_offset": 21,
      "end_offset": 22,
      "type": "word",
      "position": 9
    },
    {
      "token": "을",
      "start_offset": 22,
      "end_offset": 23,
      "type": "word",
      "position": 10
    },
    {
      "token": "판단",
      "start_offset": 24,
      "end_offset": 26,
      "type": "word",
      "position": 11
    },
    {
      "token": "하",
      "start_offset": 26,
      "end_offset": 27,
      "type": "word",
      "position": 12
    },
    {
      "token": "는",
      "start_offset": 27,
      "end_offset": 28,
      "type": "word",
      "position": 13
    },
    {
      "token": "기준",
      "start_offset": 29,
      "end_offset": 31,
      "type": "word",
      "position": 14
    },
    {
      "token": "이",
      "start_offset": 31,
      "end_offset": 32,
      "type": "word",
      "position": 15
    },
    {
      "token": "다",
      "start_offset": 32,
      "end_offset": 33,
      "type": "word",
      "position": 16
    }
  ]
}
```

- analyzer 에 옵션을 줘서 의미단위의 품사만 남기기
```json
put test-analyzer-nori
{
  "settings":{
    "index":{
      "analysis": {
        "tokenizer":{
          "nori_configured_tokenizer":{
            "type":"nori_tokenizer",
            "decompound_mode": "mixed",
            "discard_punctuation":"false"
          },
          "standard": {
            "type": "standard"
          }
        },
        "filter":{
          "nori_stop_filter":{
            "type":"nori_part_of_speech",
            "stoptags":[
              "E",
              "IC",
              "J",
              "MAG", 
              "MAJ", 
              "MM",
              "SP", 
              "SSC", 
              "SSO", 
              "SC", 
              "SE",
              "SF",
              "XPN", 
              "XSA", 
              "XSN", 
              "XSV",
              "VCP",
              "UNA", "NA", "VSV",
              "E", "NNB", "VX", "MAG"
              ]
          },
           "english_stemmer": {
            "type": "stemmer",
            "language": "english"
          },
          "lowercase": {
            "type": "lowercase"
          }
        }
        ,
        "analyzer":{
          "test_analyzer":{
            "type": "custom",
            "tokenizer":"nori_configured_tokenizer",
            "filter": [
              "nori_stop_filter",
              "english_stemmer"
            ]
          }
        }
      }
    }
}
}
```

- 
```json
{
  "tokens": [
    {
      "token": "토큰",
      "start_offset": 0,
      "end_offset": 2,
      "type": "word",
      "position": 0
    },
    {
      "token": "문서",
      "start_offset": 5,
      "end_offset": 7,
      "type": "word",
      "position": 4
    },
    {
      "token": "유사",
      "start_offset": 8,
      "end_offset": 10,
      "type": "word",
      "position": 6
    },
    {
      "token": "알고리즘",
      "start_offset": 12,
      "end_offset": 16,
      "type": "word",
      "position": 9
    },
    {
      "token": "유사",
      "start_offset": 19,
      "end_offset": 21,
      "type": "word",
      "position": 12
    },
    {
      "token": "판단",
      "start_offset": 24,
      "end_offset": 26,
      "type": "word",
      "position": 16
    },
    {
      "token": "기준",
      "start_offset": 29,
      "end_offset": 31,
      "type": "word",
      "position": 20
    }
  ]
}
```

### n-gram 

##### n-gram 

글자를 n개 만큼 잘라서 토큰으로 만드는 것을 말한다.

가령, "토큰이란 문서 유사도 알고리즘에서 유사성을 판단하는 기준이다." 라는 문장은 n gram 으로 tokenizing 한다하면 예시는 아래와 같다.

```json
POST /_analyze
{
  "text": "토큰이란 문서 유사도 알고리즘에서 유사성을 판단하는 기준이다.",
  "tokenizer": {
    "type": "ngram",
    "min_gram": 2,
    "max_gram": 2
  }
}

```

- 결과 
```
["토큰","큰이","이란", "란 " .... "알고","고리","리즘","즘에"....]
```

위의 결과처럼 정직하게 두글자씩 자르는 방법 알고리즘을 nGram 이라고 하고, 위처럼 두 글자식 나눈 것을 size 2 의 ngram 이라고 한다. 

ngram 은 유저가 오타를 입력해도 토큰 자체는 매칭이 된다는 장점이 있다.

가령 내가 "알고리즘" 이라고 검색을 하려했는데, "알고리ㅈㅡ" 라고 검색했다고 가정하자.
그러면 토큰은 

```
검색어: ["알고","고리","리ㅈ","ㅈㅡ"]
```

로 검색이 수행되어서, "알고","고리" 라는 토큰이 일치하므로 위의 텍스트를 검색에 성공할 수 있다. 

##### n-gram을 추천하지는 않는 이유와 대안

특정 토큰, 특히 짧은 토큰을 잘 처리 못합니다. 예를 들어, size 2인 n-gram이 적용되고 영어 문서가 많은 환경에서, 엘라스틱 서치의 약자인 "ES"를 검색한다고 해보세요. 그러면 검색 결과에 ES가 아닌 것도 올라옵니다. 소팅 스코어 옵션에 따라서는 그냥 ES, Elasticsearch 내용이 안 나올 수도 있습니다.

왜냐하면 "ES"라는 토큰은 n-gram을 적용하면 거의 모든 문서에 속할 것입니다. 예를 들어서 "classes", "esc", "messy" 이런 단어들이 모두 "es" 토큰을 해당 문서의 토큰 집합에 포함시킵니다.

"es"라는 토큰이 포함된 문서가 n-gram 적용 전과 비교하면 몇백배는 늘 수 있고 "ES"라는 토큰의 idf 스코어를 왕창 내려버립니다. 고전적인 문서 유사도 알고리즘(TF-IDF)에서는 각각의 토큰이 전체 문서 집합에서 얼마나 고유한지를 스코어링에 반영하는데, n-gram이 적용된 상태에서의 "es"라는 토큰은 전혀 고유하지 않습니다.

그래서 ES로 검색하면 Elasticsearch와 관련이 없는 글도 리스트에 나타납니다. 왜냐하면 해당 토큰의 스코어가 높지 않고 거의 모든 문서가 해당 토큰을 가지고 있기 때문입니다.

그리고 오타에 대응할 수 있는 좋은 옵션이 ngram 말고도 많아요. 

가장 추천하는 것은 상황이 허락한다면 vector 서치를 적용해서 아날라이저를 직접 튜닝하지 않는 것이고, 그게 아니라면 query 에서 fuzinness 옵션을 적용해서 편집거리로서 오타를 보정하는 것을 추천합니다. 


## doc_value 와  column based 자료형

fulltext 서치는 위에서 언급한 inverted index 와 문서 유사도 방식을 이용해 수행할 수 있으나, 실제로는 아래같은 요청도도 있습니다.

```
CASE 1: 검색어로 검색하되, 유사도 점수말고 생성일자 기준으로 정렬해주세요 

CASE 2: 특정 칼럼의 평균을 구해주세요. 

CASE 3: 특정 칼럼에서 칼럽값의 합을 구하거나, 10% 위치에 있는 값을 가져오기  

```

위의 세 요청은 지금까지 설명한 개념으로는 해결할 수 없는 문제입니다.

사실 ES 나 lucene 이 처음 만들어질땐, 고려되지 않은 기능들이 맞는데 인기가 많아지고 시간이 지나면서 위의 내용도 케어할 수 있는 자료형이 내장되기 시작했다. 

숫자처럼 정렬이 가능한 필드가 인덱싱되면  doc_value 라고 하는 자료형에 자동적으로 저장이 되고, 나중에 집계나 정렬에 관련한 요청이 들어오면 해당 자료형을 조회하게 된다.



### column based 자료형 

doc_value 가 column based 자료형이기 때문에 doc_value 를 이해하려면 column_based 자료형을 먼저 언급해야 한다. 

column based 자료형이란 


![data_example_1.png](images%2Fdata_example_1.png)


rmds 에 위와 같은 데이터가 저장이 되어 있다고 가정해봅시다.

![data_example_2.png](images%2Fdata_example_2.png)

이것을 다른곳에 저장할 때, RDMS 에 저장 한 것처럼 row 단위로 한줄씩 저장하는 것을 row-based 자료형이라 합니다. 
row-based 의 장점은 일단 직관적이고, key-value 관계에서 특정 데이터에 접근하는데 강점이 크다. (한번에 가져오면 되니까)


![data_example_3.png](images%2Fdata_example_3.png)

![data_example_4.png](images%2Fdata_example_4.png)
이게 column-based 데이터입니다. 첫 줄은 empId 를 해당 원본로우의 key 값과 함께 쭉 나열합니다.

그 다음줄은 {lastname}:{key}, 그 다음줄은 {Firstname}:{key}… 이렇게 칼럼에 속한 데이터들을 나열합니다.

위와 같은 방식은  특정 칼럼을 집계할 때, 모든 정보를 조회하지 않아도 된다는 장점이 있습니다.

따라서 집계에 대해 ( 특히 숫자 ) 훨신 빠른 성능을 보장합니다.

엘라스틱 서치에서 doc_value 라고 하는 자료형(옵션) 을 설정하면 해당 칼럼에 대한 정보는 row 생성 시, 원본과는 별도로 column-based 자료를 디스크에 별도로 저장합니다.

해당 자료는 집계에 대한 쿼리 시에 자동적으로 조회되어 쿼리의 조건으로 사용하게 됩니다.



### doc_value 사용예시 

kibana dashboard 를 만들기 위한 거의 모든 쿼리는 위의 doc_value 를 이용한다고 보면 됩니다.

아래는 es8의 커머스 샘플데이터와 kibana 샘플 대시보드 입니다.

![dash_boaed.png](images%2Fdash_boaed.png)

위의 대시보드 표현을 위해, 아래의 쿼리를 수행할 수 있습니다.
(상품 갯수 : 9000개, 상점 갯수 : 4500개)
   - 집계 함수: sum of reveue
   - 집계 함수: median spending
   - 집계 함수: transaction per day
   - 집계 함수: agg by category …

