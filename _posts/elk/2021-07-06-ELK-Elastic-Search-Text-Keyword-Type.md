---
layout: post
title: "[ELK] ElasticSearch에서 text 타입과 keyword 타입 차이"
subtitle: "fielddata, doc_values 에 대한 이해와 주의사항 / keyword 필드 검색시 주의사항 및 normalizer 적용"    
comments: true
categories : ELK
date: 2021-07-06
background: '/img/posts/mac.png'
---

elasticsearch 에서 선언이 가능한 문자열 타입에는 text, keyword 두가지가 있다.   
2.x 버전 이전에는 문자열은 string이라는 하나의 타입만 있었고 텍스트 분석 여부, 즉 
애널라이저 적용을 할 것인지 아닌지를 구분하는 설정이 있었다.   

`5.0버전 부터는 텍스트 분석 적용 여부를 text 타입과 keyword 타입으로 구분한다. 인덱스를 
생성할 때 매핑에 필드를 미리 정의하지 않으면 동적 문자열 필드가 생성 될때 text 필드와 
keyword 필드가 다중 필드로 같이 생성된다.`        

> keyword 타입에서 ignore above 옵션은 설정된 길이 이상의 문자열은 색인을 무시한다.   
> 단, source에는 남아 있기 때문에 다른 필드 값을 쿼리해서 나온 
결과로 가져오는 것은 가능하다.   

```
"line" : {
    "type" : "text",
    "fields" : {
        "keyword" : {
        "type" : "keyword",
        "ignore_above" : 256
        }
    }
}
```    



keyword type의 경우 exact 매칭에서 사용되고, text type의 경우 
anlyzed 매칭에 사용된다.   
text type의 경우 형태소 분석을 통해 field를 여러 terms로 나눠서 역인덱싱 과정을 
거치게 되고, keyword type은 그대로 역인덱싱 된다.   

text type 과 keyword type에 대해 각각 알아보고 fielddata, doc_values 사용 여부에 
대해 살펴보자.   

텍스트 분석 및 역 인덱스에 
대한 자세한 설명은 [링크](https://wonyong-jang.github.io/elk/2021/02/09/ELK-Elastic-Text-Analysis.html)를 참고하자.   

- - - 

## 1. Text type   

검색(search)이라는 것은 '어떤 문서가 이 키워드를 포함하는지가 궁금'하므로 
[역인덱스](https://wonyong-jang.github.io/elk/2021/02/09/ELK-Elastic-Text-Analysis.html)된 
정보를 통해 빠른 검색이 가능하다.   

그러나 sort, aggregation 과 같은 패턴은 '이 문서에서 
이 field value값이 무엇인지'가 관심이므로 역인덱스 정보가 아닌 
document를 key로, field정보를 담은 데이터 구조가 필요하다. 
이 데이터 구조가 fielddata라는 것이다.    

<img width="1000" alt="스크린샷 2021-07-06 오전 12 18 40" src="https://user-images.githubusercontent.com/26623547/124494060-58b44c80-ddf1-11eb-976c-8d80426f23da.png">   

text필드를 aggregation할때 fielddata를 true로 설정하지 않으면 다음과 같이 
에러가 발생한다.

```
Fielddata is disabled on text fields by default.
Set fielddata=true on [your_field_name] in order to load fielddata in memory by
```

아래와 같이 text field 기능 사용 시 fielddata 옵션을 추가해주면 된다.   

```
PUT my_index/_mapping/_doc
{
  "properties": {
    "my_field": { 
      "type":     "text",
      "fielddata": true
    }
  }
}
```

`하지만 fielddata의 경우 im memory 구조로 작동하기 때문에 
많은 heap memory를 소비하게 된다. 일단 field가 heap에 로딩되면 그것은 
segment의 lifetime동안 남아 있게된다. 따라서 비용이 높은 프로세스가 된다.`     

해당 field를 메모리에 적재하는 특징 때문에, 굉장히 많은 메모리를 소비하는 것은 
물론 문제다. 이는 Memory 부족으로 인한 OOM이나, CircuitBreakingException의 원인이 될 수 있다.   

하지만, 이러한 메모리 부족 문제는 클러스터에 노드를 추가하는 등의 수평적 확장을 통해 
해결 할 수 있다.     

`text field를 사용하게 되면 fielddata 데이터 구조를 사용할 수 있는데 
위의 설명과 같이 높은 비용 때문에 default 는 false로 되어 있다.`    

### 1-1) Fielddata Monitoring    

fielddata에 의해서 어느정도 메모리가 사용되고 있는지 모니터링 할 수 있다.   

아래는 index 레벨에서 조회한다. 각 index 별로 전체적인 fielddata 상태를 출력한다.   

```
$ curl -XGET 'localhost:9200/_stats/fielddata?fields=*&pretty'
```

아래는 클러스터 내의 각 node 별로 사용되고 있는 fielddata 상태를 출력한다.   

```
$ curl -XGET 'localhost:9200/_nodes/stats/indices/fielddata?fields=*&pretty'
```

- - - 

## 2. keyword type   

`텍스트 타입에는 standard analyzer가 사용되는 반면에 키워드 타입은 
anlyzer를 사용하지 않는다.`        

따라서 아래 예제는 keyword 타입에 analyzer를 실제로 사용하여 
텍스트 분석을 하는 것이 아니라 어떻게 저장되는지 
테스트 하기 위한 방법일 뿐이다.     

```
POST _analyze
{
  "text": "My name is Kaven",
  "analyzer": "keyword"
}
```

Output   

```
{
  "tokens" : [
    {
      "token" : "My name is Kaven",
      "start_offset" : 0,
      "end_offset" : 16,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

위의 응답 값에서 토큰이 하나로 묶여 있는 것을 확인할 수 있다.   
`즉 keyword 타입은 대소문자를 모두 구분(case sensitive)하며, 정확한 값을 입력해야만 
검색이 가능하다.`      

`keyword 타입은 analyzer를 사용하지 않는 대신 normalizer의
적용은 가능하다.`    

normalizer 는 single token 생성하는 것 외에 analyzer와 유사하다.   
`즉, normalizer는 keyword를 색인 전에 적용되며, 검색할 때 또한 적용된다.`   

> normalizer는 토크나이저는 적용할 수 없고 캐릭터 필터와 토큰 필터만 적용해서 사용이 가능하다.   


아래와 같이 keyword 타입 name필드에 [normalizer](https://www.elastic.co/guide/en/elasticsearch/reference/current/normalizer.html)를 
적용한 템플릿 예시이다.   


```
PUT _template/summary-template
{
    "index_patterns": ["summary*"],
    "settings": {
      "analysis": {
        "normalizer": {
          "lowercase_normalizer": {
            "type": "custom",
            "filter": ["lowercase"]
          }
        }
      }
    },
    "mappings" : {
      "_doc" : {
        "properties" : {
          "id" : {
            "type" : "keyword"
          },
          "name" : {
            "type" : "keyword",
            "normalizer": "lowercase_normalizer"
          },
        }
      }
    }
}
```

Output   

```
GET summary-20230503/_search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "name": "KAVEN"
        }
      }
    }
  }
}
```   

위와 같이 템플릿을 생성하여 인덱스가 생성될 때 직접 만든 lowercase normalizer가 
name 필드에 적용되도록 하였다.   

> 템플릿을 사용하지 않고, 직접 인덱스 settings에 추가하여도 된다.   

`name 필드에 lowercase normalizer를 적용하였기 때문에, 
인덱스에 소문자로 색인되며, 검색할 때도 동일하게 적용된다.`    

`즉, 대소문자 구분 없이 keyword 타입을 검색할 수 있다.`       

자세한 내용은 [링크](https://www.elastic.co/guide/en/elasticsearch/reference/current/normalizer.html)를
참고하자.     

```
POST summary-20230503/_analyze
{
  "normalizer": "lowercase_normalizer",
  "text": "KAVEN"

```  

Output

```
{
  "tokens" : [
    {
      "token" : "kaven",
      "start_offset" : 0,
      "end_offset" : 5,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

마지막으로 lowercase normalizer를 이용하여 색인할 때, 어떤 모양으로 색인되는지
확인하기 위해 analize api를 이용하여 위처럼 확인 할 수 있다.   



### 2-1) keyword 타입 검색    

keyword 타입으로 검색하기 위해서는 아래와 같이 검색이 가능하다.   

```
{
  "query": {
    "match": {
      "address.keyword": "891 Elton Street"
    }
  }
}
```

`여기서 중요한 점은 keyword 필드(.keyword)는 
일반적으로 text 필드의 sub field 이다.`   

즉 위에서 언급한 것처럼, 
    string 필드를 매핑 정보를 지정하지 않고 기본적으로 생성하게 되면, 
    text 필드와 keyword 필드 둘다 생성되어 다중 필드가 생성된다.   

`이때, keyword로 검색하기 위해 .keyword를 필드 뒤에 추가하여 검색하면 되지만, 
    다중 필드가 아닌 keyword 필드로만 구성된 필드의 경우는 검색이 되지 않는다.`     

```
"mappings" : {
    "_doc" : {
        "properties" : {
            "address" : {
               "type" : "keyword"
            }
        }
    }
}
```   

> 위와 같이 address의 mapping 정보를 keyword 타입으로만 구성할 수 있다.   

`keyword 타입으로만 구성된 필드의 경우는 .keyword를 추가하지 않고 아래와 같이 
검색하면 exact 검색을 할 수 있다.`    

```
{
  "query": {
    "match": {
      "address": "891 Elton Street"
    }
  }
}
```

### 2-2) doc values     

`keyword field에서는 fielddata의 in memory에서 동작하는 구조를 개선하여 
on-disk data structure인 doc_values 사용이 가능하다.   
doc_values는 아래와 같이 열 기반 저장소(columnar store)로 더욱 유리하게 
sort, aggregation 등을 할 수 있다.`    

doc_values는 default true이며, 이 값을 false로 하면 집계나 정렬이 불가능해진다.   

<img width="990" alt="스크린샷 2021-07-06 오전 12 18 46" src="https://user-images.githubusercontent.com/26623547/124494071-5ce06a00-ddf1-11eb-80c9-41bfa10c6ed6.png">     
 

- - - 

## 글 마무리    

keyword type과 text type은 이렇게 analyzed가 되냐 안되냐의 차이뿐 아니라 
fielddata, doc_values와 같은 데이터 구조 사용 여부도 
달라지게 되므로 적절한 data mapping과 옵션 설정이 중요하다.   


- - - 

**Reference**   

<https://esbook.kimjmin.net/07-settings-and-mappings/7.2-mappings/7.2.1>    
<https://wedul.site/502>   
<https://doubly12f.tistory.com/94>    

{% highlight ruby linenos %}

{% endhighlight %}


{%- if site.disqus.shortname -%}
    {%- include disqus.html -%}
{%- endif -%}

