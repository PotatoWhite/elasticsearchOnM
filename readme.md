Elasticsearch 설치 및 운영환경 구성
===================================

- 이 문서는 Elasticsearch의 설치 및 환경 구성을 통해 시스템 도입 및 확장 업무를 지원할 수 있게 하는 Quick Guide가 목표이다.
- 본 문서의 기준은 Elasticsearch 6.4.2 를 기준으로 작성 되었다.


* * *

Elasticsearch 소개
------------------
- License
    - Elasticsearch는 Full-Text 검색엔진으로 Apache Software의 검색엔진 프로젝트인 Lucene을 기반으로 한다.
    - Lucene은 Doug Cutting에 의해 개발되었으며, Apache License를 따른다.

    - Elasticsearch는 Shay Banon이 Lucene기반으로 만든 검색엔진이다.
    - Elasticsearch는 Apache License 2.0을 따른다.


* * *


ELK Stack
---------
- 검색엔진인 Elasticsearch와 함께 Logstash, Kibana라는 제품을 함꼐 사용하는 것을 의미함
    - Logstash와 Beats를 통해 Data 수집
    - Elasticsearch에 저장
    - Kibana를 통해 분석    
    <center><img src="https://www.elastic.co/guide/en/beats/libbeat/current/images/beats-platform.png" width="600" ></center>

* * *

Elasticsearch의 용어 및 개념 정리
-------------------------------

## Data 저장구조
### Document
 - JSON(Java Script Object Notation) 기반의 실제 Data를 의미하는 데이터를 가진 저장단위
 - Document ID를 통해 구분됨
 - 사용자가 생성이 가능하지만, 일반적으로 자동생성됨

### JSON(Java Script Object Notation)
 - JSON은 널리 사용하는 경량데이터 구조
 - Key:Value 형태
    <pre><code>
    {
        "name":"Potato",
        "address":{
            "city":"Seoul",
            "country":"ROK"
        }
    }
    </code></pre>

### INDEX
 - Document들의 모음, 여러 Document들을 하나의 Index로 적재됨
 - Index가 사전에 정의될 수도 있으나, 특정한 구조가 필요하지 않을 경우 최초 데이터 인입시 생성


### TYPE
 - Index의 파티션으로 사용함
 - RDB의 Table과 유사하게 생각할 수 있음
 - 하나의 Index에 Document를 저장할 때 Type을 분리하여 Indexing 할 수 있음
 - ES 6.x 이상은 Multi Type을 지원하지 않음




 * * *


## System 구성 구조
### Cluster
 - ES는 보통 Cluster로 구성 되면 하나 시앙의 노드로 구성됨, 사용자는 클러스터를 통해 데이터를 넣고 검색요청을 함
 - Cluster별 고유의 cluster_name과 cluster_uuid를 갖음

### Node
 - Cluster를 구성하는 ES프로세서 단위
 - Master, Data, Client, All Node로 구분
    - Master Node : 클러스터 구성의 기준, Node들의 H/C
    - Data Node : 실제 Data가 적재됨, Client의 요청에 Data를 반환
    - Client Node : 외부의 쿼리만 받는 Node로 부하 분산을 위해 사용됨(Master나 Data 또한 외부 쿼리 요청을 받을 수는 있음)
    - All Node : Master와 Data를 동시에 사용하는 노드(확장이 거의 없을 경우 사용됨)