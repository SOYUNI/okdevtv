# ELK
* Elasticsearch + Logstash + Kibana
* Elasticsearch는 Apache의 Lucene을 바탕으로 개발한 실시간 분산 검색 엔진이며, 
* Logstash는 각종 로그를 가져와 JSON형태로 만들어 Elasticsearch로 전송하고,
* Kibana는 Elasticsearch에 저장된 Data를 사용자에게 Dashboard 형태로 보여주는 솔루션이다.
![ELK Architecture](images/elk_arch.jpg)
* http://elastic.co 사이트 오픈소스 제품


## 장점
* Google Analytics(GA)의 데이터로 사이트 접속 통계를 구할 경우 원하는 대로 데이터를 획득하기 어렵다.
* 자체 서버의 모든 로그를 100% 수집할 수 있기 때문에 데이터에 대한 신뢰성이 높다.
* 파라미터 값별로 통계를 볼 수 있기 때문에 정확한 데이터 분석이 가능하다.
* 검색엔진(lucene)이 포함되어 있어, 빠르게 데이터를 검색할 수 있다.
* 모두 오픈소스이며 자유롭게 사용이 가능하다.


## 사전 준비
* 로그수집 서버(AWS 추천)
  * aws 접속 key가 있는 경우 
  * 윈도우에서 git bash 추천(http://git-scm.com). putty 접속보다 쉬움
* 리눅스 서버 CentOS 또는 Ubuntu
* Java 1.7 이상(esp. logstash는 1.8이상 필요)

## nginx 설치(샘플용)

```
sudo yum install nginx -y
sudo service nginx start
curl -i http://localhost
sudo chown -R ec2-user:ec2-user /var/log/nginx /usr/share/nginx/html
echo "<h1>Hello World</h1>" > /usr/share/nginx/html/hello.html
```

* http://아이피/
* http://아이피/hello.html

## AWS 포트 설정
* EC2 Security Groups
* 외부 접근 포트 추가(inbound)
  * http(80)
  * elasticsearch(9200)
  * kibana(5601)


## 설치
* Elasticsearch 
* Kibana
* Logstash (FluentD로 대치 가능)

* 버전을 맞춰서 작업하는 것이 좋지만, 최신 버전으로 작업해도 동작함(2015/10/20 현재)
* 설치 위치 /opt/ 또는 ~/local/ 권장


## Elasticsearch 설치

```
mkdir ~/local
cd ~/local
wget https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-2.2.0.tar.gz
tar xvfz elasticsearch-2.2.0.tar.gz
cd elasticsearch-2.2.0
vi config/elasticsearch.yml
`# network.host: 192.168.0.1`의 주석을 풀고 `network.host: 0.0.0.0`으로 변경
bin/elasticsearch
```

* 실행 확인
```
curl -i http://localhost:9200/
```

## Kibana 설치

```
cd ~/local
wget https://download.elastic.co/kibana/kibana/kibana-4.4.0-linux-x64.tar.gz
tar xvfz kibana-4.4.0-linux-x64.tar.gz
cd kibana-4.4.0-linux-x64
```

* elasticsearch 연결
  * `config/kibana.yml` 파일을 열어서 `elasticsearch_url` 값을 맞춰줌

```
bin/kibana
```

* 실행 확인
http://yourhost.com:5601


## Logstash 설치

```
cd ~/local
wget https://download.elastic.co/logstash/logstash/logstash-2.2.0.tar.gz
tar xvfz logstash-2.2.0.tar.gz
cd logstash-2.2.0
```

* conf 파일 생성

```
mkdir logconf
vi logconf/nginx.conf
```

logconf/nginx.conf
```
input {
    file {
        path => "/var/log/nginx/access.log"
        start_position => beginning
    }
}
filter {
    grok {
        match => { "message" => "%{COMBINEDAPACHELOG}"}
    }
    geoip {
        source => "clientip"
    }
}
output {
    elasticsearch {}
    stdout {}
}
```

* logstash 실행 
```
# test
bin/logstash -f logconf/nginx.conf -t
# run
bin/logstash -f logconf/nginx.conf
```


## Kibana 통계

### 시각화
  * 테이블
  * 차트
  

### 대시보드 만들기



## part 2

### Logstash
* 파라미터 필드 만들기
```
filter {
    mutate {
        add_field => {
            "tmp" => "%{request}"
        }
    }
    if [tmp] =~ "\?" {
        mutate {
            split => [
                "tmp", "?"
            ]
            add_field => {
                "params" => "%{[tmp][1]}"
            }
        }
        kv {
            field_split => "&"
            source => "params"
            include_keys => [ "category", "utm_source" ]
            prefix => "param_"
        }
    }
}
```

* 이미지 제거
```
filter {
    if [message] =~ "^#|\.(css|js|ico|png|xml|jpg|JPG|gif|jpeg|eot\?) " {
        drop {}
    }
}
```


### Kibana
* 질의어 문법(query syntax)

* request: "uri"


## Filebeat
* (Optional)
* logstash forwarder 의 경량(lightweight) 버전

```
cd ~/local
wget https://download.elastic.co/beats/filebeat/filebeat-1.0.1-x86_64.tar.gz
tar xvfz filebeat-1.0.1-x86_64.tar.gz
cd filebeat-1.0.1-x86_64
./filebeat -e -c filebeat.yml
```

## 참고
* ELKR (ElasticSearch + Logstash + Kibana + Redis) 를 이용한 로그분석 환경 구축하기
  * http://brantiffy.axisj.com/archives/418

* EMOCON 2015 F/W ELK 스택을 사용한 서울시 지하철 대시보드 만들기 *추천*
  * https://youtu.be/ec-XzM6_CgU

* ELK 구축하기 1 – LOGSTASH
  * http://linux.systemv.pe.kr/elk-구축하기-1-logstash/

* [Ubuntu] ELK 설치 및 테스트 하기
  * http://digndig.kr/ubuntu/449/

* Splunk 대체 Solution으로서의 ELK Stack 
  * http://blog.embian.com/18

* How To Install Elasticsearch, Logstash, and Kibana 4 on Ubuntu 14.04
  * https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-4-on-ubuntu-14-04

* ELK 프로그래밍 방송 영상
  * http://bit.ly/okdevtv-elk

* Logstash Configuration
  * https://www.elastic.co/guide/en/logstash/current/event-dependent-configuration.html

* Elasticsearch(Lucene) Query Syntax
  * https://lucene.apache.org/core/2_9_4/queryparsersyntax.html
  

