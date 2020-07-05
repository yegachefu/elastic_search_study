# Elasticsearch 3장

1. 색인 시, 문서의 데이터 유형에 따라 필드에 데이터타입을 지정하는 것을 매핑이라고 한다.

매핑을 설정하지 않으면 엘라스틱서치가 자동으로 설정하지만, 실제 운영환경에서는 문제를 일으킬 수 있기 때문에 권장하지 않는다.

매핑정보를 설정할 때, 고민해야할 사항들

- 문자열을 분석할 것인가
- _source에 어떤 필드를 정의할 것인가
- 날짜 필드를 가지는 필드는 무엇인가
- 매핑에 정의되지 않고 유입되는 필드는 어떻게 처리할 것인가

교재대로 인덱스를 생성하면 에러 발생

```java
{
    "error": {
        "root_cause": [
            {
                "type": "illegal_argument_exception",
                "reason": "The mapping definition cannot be nested under a type [_doc] unless include_type_name is set to true."
            }
        ],
        "type": "illegal_argument_exception",
        "reason": "The mapping definition cannot be nested under a type [_doc] unless include_type_name is set to true."
    },
    "status": 400
}
```

→ _doc를 빼고 생성하도록 한다.

[2. 매핑 파라미터 정리](https://www.notion.so/73e53ae2e3dc47f1adf883779bd55da4)

[3. 메타 필드 정리](https://www.notion.so/81d7a8f576da40cd87b86baaa51f090e)

[4. 필드 데이터 타입](https://www.notion.so/d5794d6538ed40f9967e1cb51719e7fe)

3.4 엘라스틱서치 분석기

3.4.1 엘라스틱서치는, 다양한 분석기를 제공하는 루씬을 활용하여 텍스트 검색엔진을 제공한다.

```jsx
POST _analyze
{
	"analyzer": "standard",
	"text": "우리나라가 좋은나라, 대한민국 화이팅"
}
```

```jsx
{
	"tokens": [
		{
			"token": "우리나라가",
			"start_offset": 0,
			"end_offset": 5,
			"type": "<HANGUL>",
			"position": 0
		},
		{
			"token": "좋은나라",
			"start_offset": 6,
			"end_offset": 10,
			"type": "<HANGUL>",
			"position": 1
		},
		{
			"token": "대한민국",
			"start_offset": 12,
			"end_offset": 16,
			"type": "<HANGUL>",
			"position": 2
		},
		{
			"token": "화이팅",
			"start_offset": 17,
			"end_offset": 20,
			"type": "<HANGUL>",
			"position": 3
		}
	]
}
```

위와 같이, "우리나라가 좋은나라, 대한민국 화이팅"이라는 문장에서 "대한민국"을 검색하면 검색이 되지 않는다. 이유는 엘라스틱서치의 분석기를 통해 분석된 데이터에는 "대한민국"이라는 단어가 포함되어있지 않기 때문이다.

이러한 분석기는 언어별로 조금씩 상이하며, 원하는 분석기를 Customizing 할 수도 있다.

3.4.2 역색인 구조

책의 마지막 부분에 나열된 목록이 있다. 단어와 페이지가 열거되어있어서 단어가 등장하는 페이지를 쉽게 찾아볼 수 있다. 이처럼, 루씬도 역색인이라는 특수한 방식으로 구조화돼 있다.

1. 해당 단어가 어떤 문서에 속해 있는지에 대한 정보

역색인 구조는 아래와 같다.

1. 모든 문서가 가지는 단어의 고유 단어 목록
2. 전체 문서에 각 단어가 몇 개 들어가있는지에 대한 정보
3. 하나의 문서에 단어가 몇 번씩 출현했는지에 대한 빈도

```jsx
문서1
	elasticsearch is cool
문서2
	Elasticsearch is great
```

문서의 역색인을 위해 토큰화 과정이 필요하다.

[토큰 정보](https://www.notion.so/a0a72fb7f37e480eaa430b8714a7b79a)

> Question. "elasticsearch"을 검색어로 지정하면 어떻게 될까?
expect : 문서1과 문서2
actual : 문서1
solution : 가장 간단한 방법은 소문자로 변환

[변환된 토큰 정보](https://www.notion.so/1c608875cd8d464bb9e88bc7d865d1ba)

색인을 한다는 것은, 역색인 파일을 만든다는 것이지 원문 자체를 변경한다는 의미는 아니므로 색인에 들어갈 토큰만 변경되어 저장되고 실제 문서의 내용은 변함이 없다

3.4.3 분석기 구조

분석기의 프로세스는 다음과 같다.

1. 문장을 특정한 규칙에 의해 수정 (Character Filter)
    - 입력된 텍스트에 대해, 문장 분석 전 특정 단어를 변경하거나 HTML과 같은 태그를 제거
    - 텍스트를 개별 토큰화하기 전의 전처리 과정
    - ReplaceAll() 패턴 혹은 사용자가 정의한 필터를 통해 변경
2. 수정한 문장을 개별 토큰으로 분리 (Tokenizer Filter)
    - 텍스트를 어떻게 나눌 것인가를 정의
    - 언어에 따라 적절한 Tokenizer 사용 가능
3. 개별 토큰을 특정한 규칙에 의해 변경 (Token Filter)
    - 토큰화된 단어를 하나씩 필터링하여 사용자가 원하는 토큰으로 반환
    - 여러 단계가 순차적으로 이뤄지며, 어떻게 순서를 지정하냐에 따라 검색의 질이 달라짐

    ```jsx
    <B>Elasticsearch</B> is cool
    ```

    ```jsx
    {
    	"settings": {
    	"index" : {
    		"number_of_shards": 5,
    		"number_of_replicas": 1
    		}
    	},
    	"analysis": {
    		"analyzer": {
    			"custom_movie_analyzer": {
    				"type": "custom",
    				**"char_filter"**: [
    					"html_strip"
    				],
    				**"tokenizer"**: "standard",
    				**"filter":** [
    					"lowercase"
    				]
    			}
    		}
    	}
    }
    ```

    위의 분석기가 수행되는 단계를 살펴보면 다음과 같다.

    ```jsx
    <B>Elasticsearch</B> is cool
    ```

    [Character Filter](https://www.notion.so/Character-Filter-b29ff541488848e7916b28e917efaeb0)

    ```jsx
    Elasticsearch is cool
    ```

    [Tokenizer](https://www.notion.so/Tokenizer-91ec4a767d164a94b55203d16fe2634c)

    ```jsx
    Elasticsearch -> Token1, Position 1
    is -> Token 2, Position 2
    cool -> Token 3, Position 3
    ```

    [Token Filter](https://www.notion.so/Token-Filter-ff23c64906a54a03a7a58a5ad57013db)

    ```jsx
    elasticsearch -> Token1, Position 1
    is -> Token 2, Position 2
    cool -> Token 3, Position 3
    ```

    ```jsx
    {
    	"field": "title",
    	"text": "캐리비안의 해적"
    }
    ```

    ```jsx
    {
    	"settings": {
    		"analyzer": {
    			"movie_lower_test_analyzer":{
    				"type": "custom",
    				"tokenizer": "standard",
    				"fileter": [
    					"lowercase"
    				]
    			},
    			"movie_stop_test_analyzer":{
    				"type": "custom",
    				"tokenizer": "standard",
    				"fileter": [
    					"lowercase",
    					"english_stop"
    				]
    			}
    		}
    	}
    }
    ```

    ```jsx
    {
    	"mappings": {
    		"properties": {
    			"title":{
    				"type": "text",
    				"analyzer": "movie_stop_test_analyzer",
    				"search_analyzer": "movie_lower_test_analyzer"
    			}
    		}
    	}
    }
    ```

    이와 같이, 색인과 검색 시에 분석기를 각각 달리 설정할 수 있다. 

    분석기를 매핑할 때에는 기본적으로 "analyzer"라는 항목으로 설정하게 되는데 이는 색인과 검색 시점 모두 동일한 분석기를 사용하겠다는 뜻이다. 각 시점에 다른 분석기를 사용하려면 "search_analyzer"를 재 정의하면 된다.

    3.4.3.2 대표적인 분석기

    Standard Analyzer

    - 별다른 정의 없이, 필드의 데이터 타입을 Text타입으로 사용한다면 Default로 사용하게 되는 분석기
    - 공백 혹은 특수 기호를 기준으로 토큰 분리
    - 모든 문자를 소문자로 변경하는 토큰 필터 사용

    [파라미터](https://www.notion.so/3d997ff1d5a74171a85df4c1479b4cbc)

    Whitespace

    - 공백 문자열을 기준으로 토큰을 분리

    Keyword

    - 전체 입력 문자열을 하나의 키워드처럼 처리하며, 토큰화 작업을 하지 않는다.

    3.4.4 전처리 필터

    분석기의 핵심이 진행되기 전, 데이터를 정제하는 수행하는 단계이다. 하지만, 토크나이저 내부에서도 일종의 전처리가 가능하기에, 전처리 필터는 상대적으로 활용도가 떨어진다.

    Html_strip_char 필터 (escaped_tags) : HTML의 태그를 전부 삭제한다.

    3.4.5 토크나이저 필터

    분석기를 구성하는 가장 핵심적인 요소이며, 분석기에서 어떤 토크나이저를 사용하냐에 따라 전체적인 성격이 결정된다. 

    Standard 토크나이저

    일반적으로 사용되는 토크나이저이며, 대부분의 기호를 만나면 토큰으로 나눈다.

    [파라미터](https://www.notion.so/70a7d2d420474e0db79680d521b7c365)

    Whitespace 토크나이저

    공백을 만나면 텍스트를 토큰화한다.

    [파라미터](https://www.notion.so/b8588bcd52c54ad18f811c9066a9a334)

    Ngram 토크나이저

    한 글자씩 토큰화 하는 것이 기본이며, 특정 문자 지정도 가능하다. 그 밖에도 다양한 옵션을 조합해서 자동완성 등의 기능에 활용할 수 있다.

    [파라미터](https://www.notion.so/603a92b541c044bca89b1af001a8505b)

    Edge Ngram 토크나이저

    지정된 문자의 목록 중 하나를 만날 때마다, 시작 부분을 고정시켜 단어를 자르는 방식으로 사용하는 토크나이저로서, 자동완성 등에 활용이 가능하다.

    [파라미터](https://www.notion.so/bc683bd51b1e4b3ba3d4918fe81bb43f)

    Keyword 토크나이저

    텍스트를 하나의 토큰으로 만든다.

    [파라미터](https://www.notion.so/67d988c25a3e48a2ab433c4369bba724)

    3.4.6 토큰 필터

    토크나이저로부터 분리된 토큰을 변형하거나 추가, 삭제할 때 사용하는 필터

    분리된 토큰은 배열 형태로 저장되어 토큰 필터로 전달되며, 독립적으로 사용될 수는 없다.

    Ascii Folding 토큰 필터

    아스키 코드에 해당하는 127개의 문자에 해당하지 않는 경우, 문자를 ASCII 요소로 변경

    ```jsx
    {
    	"filter": [
    		"asciifolding"
    	]
    }
    ```

    Lowercase 토큰 필터

    전체 문자열을 소문자로 변환

    ```jsx
    {
    	"filter": [
    		"lowercase"
    	]
    }
    ```

    Uppercase 토큰 필터

    전체 문자열을 대문자로 변환

    ```jsx
    {
    	"filter": [
    		"uppercase"
    	]
    }
    ```

    Stop 토큰 필터

    불용어로 등록할 사전을 구축해서 사용하는 필터로, 인덱스로 만들고 싶지 않거나 검색되지 않게 하고 싶을 때 사용

    [파라미터](https://www.notion.so/32abd7589f2c4a928266cff6534c32b0)

    Stemmer 토큰 필터

    Stemming 알고리즘을 사용해 토큰을 변형하는 필터로서, 여러 형태를 지원

    [파라미터](https://www.notion.so/0032a8ecf4394e3296d4ebfefb4035d0)

    Synonym 토큰 필터

    동의어를 처리할 수 있는 필터

    [파라미터](https://www.notion.so/8e7ad3eda1254dc7b8e307ce3c1e92ac)

    Trim 토큰 필터

    앞, 뒤 공백을 제거하는 토큰 필터

    3.4.7 동의어 사전

    - 검색 기능을 보다 여러가지 방향으로 활용할 수 있게 도와주는 도구 중 하나로서, 원문에 특정 단어가 존재하지 않더라도, 단어에 대한 동의어나 유의어, 유사어 등을 매칭시켜 검색할 수 있게 해주는 기술이다.

        ex) "elasticsearch"라는 원문이 등록되어 있고, "엘라스틱서치"로 검색했을 때 검색이 가능하게 함

    - 직접 등록은 실무에서 잘 사용하지 않고, 사전을 파일 형태로 등록해서 관리하는 방식을 사용한다.
    - 동의어 사전은 실시간으로 적용되지 않으므로, 등록 후 적용 시 Reload해야 한다.
    - 동의어 사전은 색인 시점 / 검색 시점 모두에서 사용 될 수 있다.
    - 검색 시점에는 사전의 내용이 변경되어도 반영이 되지만, 색인 시점에서 사전의 내용이 변경되면 반영되지 않는다. 이 경우, **기존 색인을 모두 삭제 후 색인을 다시 생성하는 방법**과 **색인 시점에는 적용하지 않고 검색 시점에만 적용하는 방식**으로 문제를 해결한다.

    3.4.7.1 동의어 추가

    동의어 추가 시에는 단어를 쉼표(,)로 분리해 등록한다.

    최신 버전의 ES에서는 토큰 분리 후 대소문자 구별하지 않는다.

    ex)
    "Elasticsearch"와 "엘라스틱서치"를 동의어로 지정하고 싶다

    : "Elasticsearch,엘라스틱서치"

    3.4.7.2 동의어 치환

    특정 단어를 어떤 단어로 변경 시에 사용하며, 화살표(⇒)로 등록한다.

    ex)

    "Harry"를 "해리"로 변경하고 싶다

    : "Harry ⇒ 해리"

    3.5 Document API 이해하기

    문서생성

    문서 생성 시, 기본적으로 ID가 필요하며, 지정하지 않으면 자동으로 ID가 부여된다.

    문서버전

    색인된 모든 문서는 버전을 가지고 있으며, 문서의 변경이 일어날 때마다 버전 값이 증가한다. 버전 값 증가 이후에 이전 버전 값을 조회하면 에러 반환

    오퍼레이션 타입

    같은 ID로 색인이 반복적으로 일어날 경우, UPDATE를 매번 호출하게 된다. 이와 같은 사항을 방지하기 위해, 동일한 요청이 들어올 때 색인 실패처리를 할 수 있다. 요청 파라미터에 op_type=create 추가

    타임아웃 설정

    대기시간을 조정할 수 있다. 요청 파라미터에 timeout={{minute}} 추가

    GET API 요청 시, 특정 필드 제외

    _source의 모든 필드를 select하기 버거울 때, 특정 필드를 제외할 수 있다. 요청 파라미터에 _source_exclude={{필드명}} 추가

    Delete API 요청

    Delete 요청 시, 인덱스를 삭제하면 모든 문서가 삭제되므로 주의
    
    
    
notion : https://www.notion.so/ipaddress/Elasticsearch-3-ca8d208c9dba4a9cb6db316b20b54c35