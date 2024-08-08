

# 개요

Elasticsearch 의 기능과 사용처에 대해 알아봅니다.

각 기능의 제공되는 원리와 어떻게 사용법과 연결되는지 알아봅니다.

## 예상 독자

- ES 에 입문하기 전, ES 가 어떤 기능을 할 수 있는지 알고 싶은 독자
- ES 를 사용하고 있지만 빠른 검색 속도, indexing 속도를 어떻게 보장하는지 알고 싶은 독자
- ES 튜닝 문서를 읽기를 시도해 보았지만, 무슨 뜻인지 아직 이해가 되지 않는 독자.
- dankim 깃허브의 ES 시리즈를 처음부터 읽어볼 독자

## 예상 독자가 아닌 사람

- ES 의 API 호출 방법과 각 파라미터의 사용법을 알고 싶은 독자
    - 공식문서가 더 잘 설명해 줄 것입니다. 이 문서는 사용법보단 원론적인 내용에 가깝습니다.
    - 공식문서 링크 : https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
  

- 튜닝이나 원리보다는 지금 당장 빠른 시간 안에 프로덕션에 적용해서 기능을 만들어야 하는 독자
    - 한국어 문서 중 빠르게 적용하기 좋은 문서가 있습니다 : https://esbook.kimjmin.net/

---

# 목차
- [ES 의 기능](#es-의-기능)
  - [Fulltext search ](#fulltext-search--inverted-index)
  - [Inverted Index](#invereted-index-)
  - [column based 자료구조, doc_value 를 통한 집계](#column-based-자료구조-doc_value-를-통한-집계)
  - [vector search](#vector-search)
  - [그 외 기능들](#그-외-기능들)
    - [분산 처리에 대한 지원](#분산-처리에-대한-지원)
- [ES 의 사용처](#es-의-사용처)
  - [검색 서비스로서의 ES](#검색-서비스로서의-es)
  - [로그 분석 시스템으로서의 ES](#로그-분석-시스템으로서의-es)


---

# ES 의 기능


## Fulltext search
Fulltext search는 검색어에 대해서 문서집합에서 **유사한 문서**를 찾아주는 기술입니다.
이 챕터에서는  fulltext 서치의 개념과 어떻게 문서를 유사한 순서대로 줄 세우는지에 대해 알아봅니다. 

### 문서가 유사한지 판단하는 기준

문서가 얼마나 유사한지에 대한 정의는 시대에 따라서 조금씩 달라지지만 크게 두 가지 관점으로 나뉩니다.

1. 문서를 토큰으로 나눈 결과를 문서 유사도 알고리즘 ( 겹치는 수, tf-idf, bm25) 의 결과로 순서대로 줄 세운 것.
2. 문서를 sentence transformer 모델로 변환한 뒤 벡터간 거리가 가까운 순으로 줄 세우는 것.

고전적인 의미에서의 검색은 (1) **문서유사도**와 **문서의 토큰화** 이것만 말하는 것이였으나 2023년 들어서부터는 (2) 벡터 검색에 대한 관심이 크게 커지고 있습니다.

벡터 검색을 간단하게 설명하면 문서의 string 본문을 vector 로 치환을 한 다음, 이 벡터간의 유사한 정도를 나열하는 방식입니다. 텍스트를 직접 로직으로 조작할때는 잡을 수 없는 케이스가 벡터상에서는 가까운 위치에 있기 때문에 쉽게 양질의 검색 서비스를 만들 수 있습니다.

가령, "데이터베이스"와 "database" 는 문자의 비교로 봤을 땐, 유사한점이 없지만 벡터상에서는 가까운 위치에 위치하기 때문에 "database" 로 검색을 해도 "데이터베이스" 가 포함된 문서를 가져올 수 있습니다.


### 문서유사도 알고리즘과 TF-IDF

##### 문서 유사도란 
문서가 얼마나 유사한지 판단하여 숫자로 도출하는 알고리즘이다.
가령, 겹치는 단어가 있을 때마다 +1 로 계산하는 문서 유사도 알고리즘이 있다고 치면 아래의 문서를 찾을 때 "바나나" 라는 검색어에 대해 
A 는 2점, B 는 1점으로 A 를 반환해주면 되는 것이다.

```
A = "바나나 바나나"
B = "딸기 바나나"
```


##### TF-IDF 란
실제로는 이것보다 복잡한 방법을 사용하는데, 대표적인 방법은 TF-IDF 라는 방법이다. TF-IDF 는 전체문서에서 **언급된 횟수가 많을수록 해당 단어의 가중치를 줄이는 방법이다.**

실제로는 단어 기준으로 나누진 않고 토큰이라고 하는 개념으로 문장을 나눈다. 토큰의 개념은 다음 챕터에서 다룬다. 이 장에선 "토큰"을 문장의 의미단위 정도로 이해해 주기를 바란다.  

이해를 위해 아래에 토큰별 가중치를 달리하지 않은 예시를 첨부했다.

```json
문서 A : "쿠버네티스란 ..... 이러 저러 하다"
문서 B : "이거는 내가 어제 산거고, 이거는 우리집에 원래 있던거야 "
문서 C : "이것(이거)은 큰 발견입니다."

검색어 : "쿠버네티스 이거 좋은건가요?"
```

토큰(단어)의 스코어가 모두 같다면 B 가 1위다. **"이것", "이", "그" 등의 많은 문서에서 언급되는 단어(토큰)은 의도적으로 가중치를 줄여야한다.**
위의 예시처럼 A문서 ("쿠버네티스")를 최상단으로 올리려고 의도적으로 체점 기준을 바꾼것이 TF-IDF 이다.

TF란 해당 document 내부에서 언급된 횟수, IDF 란 전체 document 에서 언급된 비율로 나누는 것을 말한다. tf-idf 밸류는 tf 값과 idf 값을 곱한 것을 말한다. 

즉, 각 어떤 단어(토큰)에 대해 도큐먼트에서 언급된 횟수가 높을수록 스코어가 올라가고 전체 도큐먼트에서 언급된 횟수가 많을수록 떨어진다.

식으로 나타내면 아래와 같다.

```json
TF =  {document 내 해당 토큰이 언급된 수} / {document 내 전체 토큰 수}
IDF = Log( {전체 문서 수} / ( 1 + {토큰을 포함하는 문서 수}))
TF-IDF = 각 토큰의 TF*IDF 를 누계한 것 
```

위와 같은 방법으로 어떤 문서간의 유사도를 비교할 때, 많이 언급되는 단어(토큰)은 의도적으로 점수를 낮춰버릴 수 있다.
주로 "조사", 영어로 따지자면 "be 동사" 이런 것들의 스코어를 낮추고, "고유명사" 등의 토큰의 스코어를 상대적으로 올리는 방법이라고 볼 수 있다.

위의 TF 식과 ,IDF 식은 고정적인 것이 아니며 해당 문서에서의 토큰 비중, 전체 문서에서의 토큰 비중을 고루 비교하여 스코어를 매기는 방법을 뭉뚱그려서 TF-IDF 라고 말하기도 한다 .

##### BM25

TF-IDF 방법의 일종으로, Elasticsearch 의 default 문서유사도 알고리즘이다. 일반적인 TF-IDF 와의 차이점은 아래와 같다.


1. 특정 토큰의 가중치를 선형적으로 증가시키지 않고, 그보다 느리게 증가시킵니다.
2. TF 계산 시 커스터마이징 가능한 파라미터를 추가하여, 가중치 증가 속도를 조절할 수 있습니다.

```json
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


### Token

토큰이란 문서 유사도 알고리즘에서 유사성을 판단하는 기준이다.

문서를 token으로 쪼게는 것을 **Tokenizing** 이라고 한다.

토큰을 도출해내는 방식은 그때그때 개발자가 결정한다. 가령 띄어쓰기를 기준으로 나누기로 했다면 아래처럼 나누어진다.

```json
INPUT = "토큰이란 문서 유사도 알고리즘에서 유사성을 판단하는 기준이다."
TOKENS = ["토큰이란", "문서", "유사도", "알고리즘에서", "유사성을", "판단하는", "기준이다."]
```

##### Analyzing 

위처럼 단순히 띄어쓰기로 나누는 것은 검색서비스를 만들기에는 불충분하다. 왜냐하면 "토큰" 이라는 검색어로 "토큰이란" 이라는 토큰과 연관성을 찾을 수 없기 때문에 더 세심한 처리가 필요하다.

가령, 형태소 분석기를 이용해 단어에서 쓸데없는 조사를 빼버리거나, n-gram 을 통해서 오타가 있어도 검색이 가능한 형태를 만드는 것을 말한다.

elasticsearch 에서는 동의어 사전을 추가해, 어떤 token 에 대해서 동의어에 해당하는 token을 추가하는 것도 여기에 속한다.

가령, nori analyzer 로 위의 인풋을 나눈 결과는 아래와 같다.

```json
["토큰", "이", "란", "문서", "유사", "도", "알고리즘", "에서", "유사", "성", "을", "판단", "하", "는", "기준", "이", "다"]
```

보통, 모든 품사의 token 이 필요하지는 않기 때문에 stop word 를 조정하여 의미 단위의 단어만 토큰으로 만든다. 아래는 해당 analzyer 의 예시이다.

- analyzer setting example
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

위의 stopword 를 지정한 analyzer 로 쪼갤 시 아래와 같이 나뉜다

```json
INPUT = "토큰이란 문서 유사도 알고리즘에서 유사성을 판단하는 기준이다."
TOKENS = ["토큰", "문서", "유사", "알고리즘", "유사", "판단", "기준"]
```
- analyze api 
```shell
post test-analyzer-nori/_analyze
{
  "analyzer" :"test_analyzer",
  "text":"토큰이란 문서 유사도 알고리즘에서 유사성을 판단하는 기준이다."
}
```


## Invereted index 

### 간단한 inverted index 예시 

### 빈도수가 고려된 inverted index 예시 

### 압축 공간을 고려한 trie 를 적용한 inverted index 예시 

## vector search

이 부분은 내용이 많아서 별도 문서로 뽑았습니다 :) 

[벡터서치 소개와 고전적인 검색과의 비교](https://github.com/dkGithup2022/es_vector_search_and_comparison_of_bm25_algorithm_search)


##  doc_value

### column_based 자료구조



