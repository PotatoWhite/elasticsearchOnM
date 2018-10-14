Elasticsearch 설치 및 운영환경 구성
===================================

- 이 문서는 Elasticsearch의 설치 및 환경 구성을 통해 시스템 도입 및 확장 업무를 지원할 수 있게 하는 Quick Guide가 목표이다.
- 본 문서의 기준은 Elasticsearch 6.4.2 를 기준으로 작성 되었다.



Elasticsearch 소개
------------------
- License
    - Elasticsearch는 Full-Text 검색엔진으로 Apache Software의 검색엔진 프로젝트인 Lucene을 기반으로 한다.
    - Lucene은 Doug Cutting에 의해 개발되었으며, Apache License를 따른다.

    - Elasticsearch는 Shay Banon이 Lucene기반으로 만든 검색엔진이다.
    - Elasticsearch는 Apache License 2.0을 따른다.


ELK Stack
---------
- 검색엔진인 Elasticsearch와 함께 Logstash, Kibana라는 제품을 함꼐 사용하는 것을 의미함
    - Logstash와 Beats를 통해 Data 수집
    - Elasticsearch에 저장
    - Kibana를 통해 분석

![ELK Stack][https://www.elastic.co/guide/en/beats/libbeat/current/images/beats-platform.png]
