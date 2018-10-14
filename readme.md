* * *
# Elasticsearch 설치 및 운영환경 구성

- 이 문서는 Elasticsearch의 설치 및 환경 구성을 통해 시스템 도입 및 확장 업무를 지원할 수 있게 하는 Quick Guide가 목표이다.
- 본 문서의 기준은 Elasticsearch 6.4.2 를 기준으로 작성 되었다.  



## Elasticsearch 소개
- License
  - Elasticsearch는 Full-Text 검색엔진으로 Apache Software의 검색엔진 프로젝트인 Lucene을 기반으로 한다.
  - Lucene은 Doug Cutting에 의해 개발되었으며, Apache License를 따른다.  
  - Elasticsearch는 Shay Banon이 Lucene기반으로 만든 검색엔진이다.
  - Elasticsearch는 Apache License 2.0을 따른다.  

  


------------------
## ELK Stack
- 검색엔진인 Elasticsearch와 함께 Logstash, Kibana라는 제품을 함꼐 사용하는 것을 의미함
    - Logstash와 Beats를 통해 Data 수집
    - Elasticsearch에 저장
    - Kibana를 통해 분석    
    <center><img src="https://www.elastic.co/guide/en/beats/libbeat/current/images/beats-platform.png" width="600" ></center>

  
    
------------------
# Elasticsearch의 용어 및 개념 정리  

## Data 저장구조
### 1. Document
 - JSON(Java Script Object Notation) 기반의 실제 Data를 의미하는 데이터를 가진 저장단위
 - Document ID를 통해 구분됨
 - 사용자가 생성이 가능하지만, 일반적으로 자동생성됨

### 2. JSON(Java Script Object Notation)
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

### 3. INDEX
 - Document들의 모음, 여러 Document들을 하나의 Index로 적재됨
 - Index가 사전에 정의될 수도 있으나, 특정한 구조가 필요하지 않을 경우 최초 데이터 인입시 생성


### 4. TYPE
 - Index의 파티션으로 사용함
 - RDB의 Table과 유사하게 생각할 수 있음
 - 하나의 Index에 Document를 저장할 때 Type을 분리하여 Indexing 할 수 있음
 - ES 6.x 이상은 Multi Type을 지원하지 않음



----------------------
## Cluster의 구성
- 외부에서 내부로 들여다보면 Cluster -> Node -> Shard -> Segment 로 구성됨 
<center><img src="https://raw.githubusercontent.com/exo-addons/exo-es-search/master/doc/images/image_05.png" width="600"/></center>

### 1. Cluster
 - ES는 보통 Cluster로 구성 되면 하나 시앙의 노드로 구성됨, 사용자는 클러스터를 통해 데이터를 넣고 검색요청을 함
 - Cluster별 고유의 cluster_name과 cluster_uuid를 갖음

### 2. Node
 - Cluster를 구성하는 ES프로세서 단위
 - Master, Data, Client, All Node로 구분
    - Master Node : 클러스터 구성의 기준, Node들의 H/C
    - Data Node : 실제 Data가 적재됨, Client의 요청에 Data를 반환
    - Client Node : 외부의 쿼리만 받는 Node로 부하 분산을 위해 사용됨(Master나 Data 또한 외부 쿼리 요청을 받을 수는 있음)
    - All Node : Master와 Data를 동시에 사용하는 노드(확장이 거의 없을 경우 사용됨)

### 3. Shard
- Index가 Data를 나누어 저장하는 단위
- Document를 실제 저장하는 단위
- 각 Elasticsearch의 Shard는 Lucene Index
- 단일 Lucene Index가 포함 할 수 있는 Document의 수는 2,147,483,519 건
- 모든 Replica Shard는 병렬로 검색할 수있으므로 고가용성 및 검색 처리확장 지원
- 주의 : 한번 지정한 Shard는 변경하지 못함

- Shard를 지정하지 않을 경우 문제점 : Scaling
    - 단일 노드의 디스크 볼륨 크기의 유한성으로 더이상 저장 할 수 없는 순간이 오게 됨
    -  단일 노드의 유한한 CPU, 혹은 Memory 자원으로 Indexing이나 Searching의 성능 저하

#### 3.1 Primary Shard
- Elasticsearch에 Document가 Indexing될 때 가장 처음 생성(원본)되는 Shard를 의미함
- Shard별 Number를 통해 몇 번째 Shard인지 구분이 됨
- 별도로 설정하지 않으면 5개의 Primary Shard를 기본으로 설정

#### 3.2 Replica Shard
- Replica Shard는 Indexing된 Primary Shard의 복제 본
- Elasticsearch에 Primary Shard가 Indexing된 후, Primary Shard가 저장된 Data Node와 다른 Node에 복제 됨
- 별도의 설정이 없으면 1개의 Replica Shard를 기본으로 설정
- Indexing시에 Primary Shard의 복제 과정이 추가되기 때문에 I/O가 두배로 발생함
- 디스크 볼륨도 실제 Document보다 두배 필요(일반적으로 가용 Disk의 40%만 사용가능)

### 4. Segment
- Shard는 Segment로 이루어짐
- Document가 Indexing될 때 System Buffer Cache 영역으로 적재됨
- 이후 내부적으로 Refresh를 거치며 Commit Point를 생성하여 검색이 가능한 상태로 전환됨
- 각 Document는 Immutable하며, Update시 신규 Document가 Replace됨 이 과정이 Segement Merge라함
- 많은 수의 Segment Merge될 경우 검색의 효율이 저하 됨


## (예시) Node 및 Shard의 동작 예시
### 시나리오
#### Case 01
1. Given: Single Node의 3개의 Shard로 클러스터 구성
2. When: Document가 늘어나서 모든 볼륨을 소진하여 모든 볼륨을 소진, 데이터 적재 불가
3. Then:
    1. Cluster에 동일한 설정의 Node 추가
    2. Elasticsearch Cluster가 일정 Shard를 새로 투입된 Node에 분배하여 두대의 
    3. 용량확장을 위해 두대 Node가 Indexing과 Query에 참여됨

#### Example (추후 이미지로 바꿀예정)
1. Given: Single Node의 3개의 Shard로 클러스터 구성
<pre><code>[Cluster]
[Node1 [P0] [P1] [P2]]
</code></pre>


2. When: Document가 늘어나서 모든 볼륨을 소진하여 모든 볼륨을 소진, 데이터 적재 불가
<pre><code>[Cluster]
[Node1 [P0] [P1] [P2]]
</code></pre>


3. Then : 기본설정은 가장 큰 Shard를 신규 Node로 옮긴다.
<pre><code>[Cluster]
[Node1      [P1] [P2]]
[Node1 [P0]          ]
</code></pre>


* * *
* * *
# Elasticsearch 설치
- Elasticsearch는 Java로 작성되었다. (JVM 필요)
- Elasticsearch 의 지원 내역
  - https://www.elastic.co/support/matrix

## 1. RPM 설치
- [참고] https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html
- RPM 설치 특징
  - elasticsearch user, group 자동생성
  - /etc/elasticsearch : config 생성
  - /etc/sysconfig : 환경변수 파일


#### 1.1 RPM Repository 등록 설치
1. 파일생성 /etc/yum.repos.d/elasticsearch.repo

    ~~~~
    > vi /etc/yum.repos.d/elasticsearch.repo
    ~~~~

    ~~~~
    /etc/yum.repos.d/elasticsearch.repo에 아래 내용 추가

    [elasticsearch-6.x]
    name=Elasticsearch repository for 6.x packages
    baseurl=https://artifacts.elastic.co/packages/6.x/yum
    gpgcheck=1
    gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
    enabled=1
    autorefresh=1
    type=rpm-md
    ~~~~

2. 설치    
    ~~~~
    > yum install elasticsearch
    ~~~~

3. 실행/중지
    ~~~~
    > service elasticsearch start
    > service elasticsearch stop
    ~~~~


#### 1.2 RPM Download하여 설치
1. 설치 파일 다운로드
    ~~~~
    > wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.2.rpm
    ~~~~
2. 파일 정상다운로드 확인
    ~~~~
    > wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.2.rpm.sha512
    > shasum -a 512 -c elasticsearch-6.4.2.rpm.sha512 
    ~~~~

3. 설치
    ~~~~
    > sudo rpm --install elasticsearch-6.4.2.rpm
    ~~~~

4. 실행/중지
    ~~~~
    > service elasticsearch start
    > service elasticsearch stop
    ~~~~
    

## 2. 소스설치 : tar, zip 기반의 설치
- [참고] https://www.elastic.co/guide/en/elasticsearch/reference/current/zip-targz.html
- root 계정 외 일반계정으로 설치 가능
- 설치 후 압축 해제한 위치에 관련 파일이 모여있음

1. 설치 파일 다운로드
    ~~~~
    >wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.2.zip
    ~~~~

2. 파일 정상다운로드 확인
    ~~~~
    > wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.2.zip.sha512
    > shasum -a 512 -c elasticsearch-6.4.2.zip.sha512 
    or
    > sha512sum -c elasticsearch-6.4.2.zip.sha512 
    ~~~~

3. 설치
    ~~~~
    > unzip elasticsearch-6.4.2.zip
    > cd elasticsearch-6.4.2/ 
    ~~~~

4. 실행/중지
    > bin/elasticsearch -d
    > kill -SIGTERM

## Elasticsearch 실행확인
### 프로세스 확인   
~~~~
    > ps -ef | grep elasticsearch
    centos   28770     1 99 08:44 pts/0    00:00:15 /bin/java -Xms1g -Xmx1g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+AlwaysPreTouch -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true -XX:-OmitStackTraceInFastThrow -Dio.netty.noUnsafe=true -Dio.netty.noKeySetOptimization=true -Dio.netty.recycler.maxCapacityPerThread=0 -Dlog4j.shutdownHookEnabled=false -Dlog4j2.disable.jmx=true -Djava.io.tmpdir=/tmp/elasticsearch.r16fIGcy -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=data -XX:ErrorFile=logs/hs_err_pid%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -XX:+PrintGCApplicationStoppedTime -Xloggc:logs/gc.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=32 -XX:GCLogFileSize=64m -Des.path.home=/home/centos/elasticsearch-6.4.2 -Des.path.conf=/home/centos/elasticsearch-6.4.2/config -Des.distribution.flavor=default -Des.distribution.type=zip -cp /home/centos/elasticsearch-6.4.2/lib/* org.elasticsearch.bootstrap.Elasticsearch -d
~~~~
        
### 어플리케이션 반응확인    
    > curl localhost:9200
    
### 로그 위치
    
    RPM 설치 : /var/log/elasticsearch/elasticsearch.log
    소스 설치 : {install path}/logs/elasticsearch.log
    

* * *
# Kibana 설치
- Elasticsearch를 이용해 분석도구를 제공함
- 예제 진행을 위해 Elasticsearch로 Query를 할 수 있는 도구로 사용 (Devtools)


## 1. RPM 설치
- [참고] https://www.elastic.co/guide/en/kibana/current/rpm.html


#### 1.1 RPM Repository 등록 설치
1. 파일생성 /etc/yum.repos.d/kibana.repo

    ~~~~
    > vi /etc/yum.repos.d/kibana.repo
    ~~~~

    ~~~~
    /etc/yum.repos.d/kibana.repo에 아래 내용 추가

    [kibana-6.x]
    name=Kibana repository for 6.x packages
    baseurl=https://artifacts.elastic.co/packages/6.x/yum
    gpgcheck=1
    gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
    enabled=1
    autorefresh=1
    type=rpm-md
    ~~~~

2. 설치    
    ~~~~
    > yum install kibana
    ~~~~

3. 실행/중지
    ~~~~
    > service kibana start
    > service kibana stop
    ~~~~    



* * *
# Elasticsearch 설정
~~~~
network.host: 0.0.0.0
http.cors.enabled: true
http.cors.allow-origin: "*"
~~~~

# Kibana 설정
~~~~
server.host: "0.0.0.0"
elasticsearch.url: "http://localhost:9200"
kibana.index: ".kibana"
~~~~