# ELK 스택 로그 모니터링 학습 가이드

## 목차
1. [ELK 스택 개요](#elk-스택-개요)
2. [구성 요소](#구성-요소)
3. [로그 모니터링 아키텍처](#로그-모니터링-아키텍처)
4. [설치 및 설정](#설치-및-설정)
5. [실습 예제](#실습-예제)
6. [모니터링 대시보드 구성](#모니터링-대시보드-구성)
7. [알림 설정](#알림-설정)
8. [성능 최적화](#성능-최적화)
9. [문제 해결](#문제-해결)
10. [참고 자료](#참고-자료)

## ELK 스택 개요

ELK 스택은 **Elasticsearch**, **Logstash**, **Kibana**의 조합으로 구성된 오픈소스 로그 분석 플랫폼입니다.

### 주요 특징
- **실시간 로그 처리**: 대용량 로그 데이터를 실시간으로 수집, 처리, 분석
- **확장성**: 수평적 확장이 가능한 분산 시스템
- **시각화**: 강력한 시각화 도구를 통한 데이터 인사이트 제공
- **검색 엔진**: 빠르고 정확한 전문 검색 기능

## 구성 요소

### 1. Elasticsearch
- **역할**: 분산 검색 및 분석 엔진
- **기능**: 
  - 로그 데이터 저장 및 인덱싱
  - RESTful API 제공
  - 실시간 검색 및 분석
- **특징**: JSON 기반 문서 저장, 샤딩을 통한 확장성

### 2. Logstash
- **역할**: 데이터 수집, 변환, 전송 파이프라인
- **기능**:
  - 다양한 소스에서 로그 수집
  - 데이터 파싱 및 필터링
  - 출력 대상으로 데이터 전송
- **파이프라인 구조**: Input → Filter → Output

### 3. Kibana
- **역할**: 데이터 시각화 및 대시보드 플랫폼
- **기능**:
  - 대화형 대시보드 생성
  - 차트 및 그래프 생성
  - 실시간 모니터링
  - 검색 인터페이스 제공

### 4. Beats (선택사항)
- **역할**: 경량 데이터 수집기
- **종류**:
  - **Filebeat**: 로그 파일 수집
  - **Metricbeat**: 시스템 메트릭 수집
  - **Packetbeat**: 네트워크 패킷 분석
  - **Winlogbeat**: Windows 이벤트 로그 수집

## 로그 모니터링 아키텍처

```
[로그 소스] → [Beats/Logstash] → [Elasticsearch] → [Kibana]
    ↓              ↓                   ↓             ↓
웹서버           데이터 수집          데이터 저장    시각화
애플리케이션     및 변환              및 검색       및 분석
시스템 로그      파이프라인          클러스터      대시보드
```

### 데이터 흐름
1. **수집**: 다양한 소스에서 로그 데이터 수집
2. **처리**: 데이터 파싱, 필터링, 변환
3. **저장**: Elasticsearch에 인덱싱
4. **분석**: Kibana를 통한 검색 및 시각화

## 설치 및 설정

### Docker를 이용한 설치

#### docker-compose.yml 예제
```yaml
version: '3.7'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.0
    container_name: logstash
    volumes:
      - ./logstash/config:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  elasticsearch-data:
```

### Logstash 설정 예제

#### logstash.conf
```ruby
input {
  beats {
    port => 5044
  }
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
  }
}

filter {
  if [fields][log_type] == "nginx" {
    grok {
      match => { "message" => "%{NGINXACCESS}" }
    }
    date {
      match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
    mutate {
      convert => { "response" => "integer" }
      convert => { "bytes" => "integer" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

## 실습 예제

### 1. 웹 서버 로그 모니터링

#### Nginx 액세스 로그 분석
```bash
# Filebeat 설정
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  fields:
    log_type: nginx
  fields_under_root: true

output.logstash:
  hosts: ["localhost:5044"]
```

#### 주요 모니터링 지표
- 응답 시간 분석
- HTTP 상태 코드 분포
- 트래픽 패턴 분석
- 에러율 모니터링

### 2. 애플리케이션 로그 모니터링

#### Spring Boot 애플리케이션 로그
```json
{
  "@timestamp": "2024-01-15T10:30:00.000Z",
  "level": "ERROR",
  "logger": "com.example.service.UserService",
  "message": "Failed to save user data",
  "thread": "http-nio-8080-exec-1",
  "exception": "java.sql.SQLException: Connection timeout"
}
```

## 모니터링 대시보드 구성

### 1. 대시보드 생성 단계
1. Kibana 접속 (http://localhost:5601)
2. **Dashboard** 메뉴 선택
3. **Create dashboard** 클릭
4. 시각화 패널 추가

### 2. 주요 시각화 유형

#### 라인 차트
- 시간별 로그 발생량
- 응답 시간 트렌드
- 에러율 변화

#### 바 차트
- HTTP 상태 코드 분포
- 사용자 에이전트 분석
- 가장 많이 접근한 페이지

#### 파이 차트
- 로그 레벨 분포
- 브라우저별 접근 통계

#### 히트맵
- 시간대별 트래픽 패턴
- 요일별 사용량 분석

### 3. 유용한 KQL 쿼리 예제

```kql
# 에러 로그만 필터링
level: ERROR

# 특정 시간 범위의 로그
@timestamp >= "2024-01-15T00:00:00" AND @timestamp <= "2024-01-15T23:59:59"

# HTTP 5xx 에러 검색
response >= 500 AND response < 600

# 특정 IP에서의 접근
client_ip: "192.168.1.100"

# 응답 시간이 느린 요청
response_time > 5000
```

## 알림 설정

### 1. Elasticsearch Watcher 설정

#### 에러율 급증 알림
```json
{
  "trigger": {
    "schedule": {
      "interval": "1m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m"
                    }
                  }
                },
                {
                  "range": {
                    "response": {
                      "gte": 500
                    }
                  }
                }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total": {
        "gt": 10
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": ["admin@example.com"],
        "subject": "높은 에러율 감지",
        "body": "지난 5분간 {{ctx.payload.hits.total}}개의 5xx 에러가 발생했습니다."
      }
    }
  }
}
```

### 2. Kibana 알림 규칙

#### 디스크 사용량 모니터링
```javascript
// 인덱스 크기 모니터링
GET _cat/indices/logs-*?v&h=index,store.size&s=store.size:desc

// 클러스터 상태 확인
GET _cluster/health
```

## 성능 최적화

### 1. Elasticsearch 최적화

#### 인덱스 설정
```json
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "refresh_interval": "30s",
    "index.mapping.total_fields.limit": 2000
  },
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "message": {
        "type": "text",
        "analyzer": "standard"
      },
      "level": {
        "type": "keyword"
      }
    }
  }
}
```

#### 인덱스 라이프사이클 관리 (ILM)
```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "10GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0
          }
        }
      },
      "delete": {
        "min_age": "30d"
      }
    }
  }
}
```

### 2. Logstash 최적화

#### 메모리 설정
```yaml
# jvm.options
-Xms2g
-Xmx2g

# logstash.yml
pipeline.workers: 4
pipeline.batch.size: 1000
pipeline.batch.delay: 50
```

#### 필터 최적화
```ruby
filter {
  # 불필요한 필드 제거
  mutate {
    remove_field => ["agent", "ecs", "host"]
  }
  
  # 조건부 처리로 성능 향상
  if [fields][log_type] == "nginx" {
    grok {
      match => { "message" => "%{NGINXACCESS}" }
    }
  }
}
```

## 문제 해결

### 1. 일반적인 문제들

#### Elasticsearch 클러스터 상태 확인
```bash
# 클러스터 상태
curl -X GET "localhost:9200/_cluster/health?pretty"

# 노드 정보
curl -X GET "localhost:9200/_cat/nodes?v"

# 인덱스 상태
curl -X GET "localhost:9200/_cat/indices?v"
```

#### 로그 수집 문제 해결
```bash
# Logstash 로그 확인
docker logs logstash

# Filebeat 상태 확인
sudo systemctl status filebeat

# 설정 파일 문법 검사
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/logstash.conf --config.test_and_exit
```

### 2. 성능 문제 해결

#### 메모리 사용량 모니터링
```bash
# JVM 힙 사용량 확인
curl -X GET "localhost:9200/_nodes/stats/jvm?pretty"

# 인덱스별 메모리 사용량
curl -X GET "localhost:9200/_cat/fielddata?v"
```

#### 디스크 공간 관리
```bash
# 오래된 인덱스 삭제
curl -X DELETE "localhost:9200/logs-2024.01.01"

# 인덱스 압축
curl -X POST "localhost:9200/logs-*/_forcemerge?max_num_segments=1"
```

## 보안 고려사항

### 1. 네트워크 보안
- Elasticsearch 포트(9200) 외부 접근 차단
- Kibana HTTPS 설정
- VPN을 통한 접근 제한

### 2. 인증 및 권한 관리
```yaml
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true

# 사용자 생성
bin/elasticsearch-users useradd kibana_user -p password -r kibana_user
```

### 3. 로그 데이터 보호
- 민감한 정보 마스킹
- 개인정보 제거 필터 설정
- 데이터 암호화

## 모니터링 체크리스트

### 일일 모니터링
- [ ] 클러스터 상태 확인
- [ ] 디스크 사용량 점검
- [ ] 에러 로그 검토
- [ ] 대시보드 정상 작동 확인

### 주간 모니터링
- [ ] 인덱스 크기 및 성능 분석
- [ ] 로그 보존 정책 점검
- [ ] 백업 상태 확인
- [ ] 알림 규칙 검토

### 월간 모니터링
- [ ] 용량 계획 수립
- [ ] 성능 트렌드 분석
- [ ] 보안 업데이트 적용
- [ ] 설정 최적화 검토

## 참고 자료

### 공식 문서
- [Elasticsearch 공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Logstash 공식 문서](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Kibana 공식 문서](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Beats 공식 문서](https://www.elastic.co/guide/en/beats/libbeat/current/index.html)

### 학습 자료
- [Elastic Stack 온라인 교육](https://www.elastic.co/training/)
- [ELK Stack 튜토리얼](https://logz.io/learn/complete-guide-elk-stack/)
- [Elasticsearch 성능 최적화 가이드](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-search-speed.html)

### 커뮤니티
- [Elastic 커뮤니티 포럼](https://discuss.elastic.co/)
- [Stack Overflow - Elasticsearch](https://stackoverflow.com/questions/tagged/elasticsearch)
- [Reddit - r/elasticsearch](https://www.reddit.com/r/elasticsearch/)

### 도구 및 플러그인
- [Cerebro](https://github.com/lmenezes/cerebro) - Elasticsearch 웹 관리 도구
- [ElastAlert](https://github.com/Yelp/elastalert) - 알림 시스템
- [Curator](https://github.com/elastic/curator) - 인덱스 관리 도구

---

이 문서는 ELK 스택을 이용한 로그 모니터링 시스템 구축과 운영에 대한 포괄적인 가이드입니다. 실제 환경에 맞게 설정을 조정하고 지속적으로 모니터링하여 최적의 성능을 유지하시기 바랍니다.