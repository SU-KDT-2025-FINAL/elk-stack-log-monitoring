# 1.2 Docker 환경 설정

## 개요

이 문서는 Docker를 활용한 ELK 스택의 실제 구축 방법을 다룹니다. Docker Compose를 사용하여 Elasticsearch, Logstash, Kibana를 통합 환경으로 구성하고, 기본적인 로그 수집 파이프라인을 구축하는 방법을 학습합니다.

## Docker 환경 구축

### 사전 요구사항

#### 시스템 요구사항
- **운영체제**: Windows 10/macOS/Linux
- **RAM**: 최소 8GB (권장 16GB)
- **디스크 공간**: 최소 20GB 여유 공간
- **Docker**: Docker Desktop 또는 Docker Engine + Docker Compose

#### Docker 설치 확인
```bash
# Docker 버전 확인
docker --version
# 출력 예: Docker version 24.0.7, build afdd53b

# Docker Compose 버전 확인
docker-compose --version
# 출력 예: Docker Compose version v2.23.0
```

### 프로젝트 구조 설정

#### 디렉터리 구조 생성
```bash
mkdir elk-stack-monitoring
cd elk-stack-monitoring

# 설정 파일을 위한 디렉터리 생성
mkdir -p logstash/config
mkdir -p logstash/patterns
mkdir -p elasticsearch/data
mkdir -p kibana/config
mkdir -p logs
```

**최종 디렉터리 구조**:
```
elk-stack-monitoring/
├── docker-compose.yml
├── .env
├── elasticsearch/
│   └── data/                # Elasticsearch 데이터 저장
├── logstash/
│   ├── config/
│   │   └── logstash.conf    # Logstash 파이프라인 설정
│   └── patterns/            # 커스텀 Grok 패턴
├── kibana/
│   └── config/
│       └── kibana.yml       # Kibana 설정 (선택사항)
└── logs/                    # 샘플 로그 파일
```

## ELK 스택 초기 설정

### Docker Compose 파일 작성

#### docker-compose.yml
```yaml
version: '3.8'

services:
  # Elasticsearch 서비스
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - cluster.name=elk-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elk-network
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Logstash 서비스  
  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    volumes:
      - ./logstash/config/logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
      - ./logstash/patterns:/usr/share/logstash/patterns:ro
      - ./logs:/usr/share/logstash/logs:ro
    ports:
      - "5044:5044"
      - "9600:9600"
    environment:
      - "LS_JAVA_OPTS=-Xms512m -Xmx512m"
    networks:
      - elk-network
    depends_on:
      elasticsearch:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9600/_node/stats || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Kibana 서비스
  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=changeme
    networks:
      - elk-network
    depends_on:
      elasticsearch:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:5601/api/status || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

# 볼륨 정의
volumes:
  elasticsearch-data:
    driver: local

# 네트워크 정의
networks:
  elk-network:
    driver: bridge
```

#### 환경 변수 파일 (.env)
```env
# ELK Stack 버전
ELASTIC_VERSION=8.11.0

# Elasticsearch 설정
ES_HEAP_SIZE=1g
ES_CLUSTER_NAME=elk-cluster

# Logstash 설정
LS_HEAP_SIZE=512m

# Kibana 설정
KIBANA_PORT=5601

# 로그 디렉터리
LOG_PATH=./logs
```

### Logstash 파이프라인 설정

#### logstash/config/logstash.conf
```ruby
# 입력 설정
input {
  # 로그 파일에서 데이터 읽기
  file {
    path => "/usr/share/logstash/logs/*.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => "plain"
  }
  
  # Beats에서 데이터 받기
  beats {
    port => 5044
  }
  
  # HTTP 입력 (테스트용)
  http {
    port => 8080
  }
}

# 필터 설정
filter {
  # 모든 로그에 타임스탬프 필드 추가
  if ![timestamp] {
    mutate {
      add_field => { "timestamp" => "%{@timestamp}" }
    }
  }
  
  # 간단한 로그 파싱 (key=value 형태)
  if [message] =~ /=/ {
    kv {
      source => "message"
      field_split => " "
      value_split => "="
    }
  }
  
  # JSON 형태 로그 파싱
  if [message] =~ /^\{.*\}$/ {
    json {
      source => "message"
    }
  }
  
  # 로그 레벨 정규화
  if [level] {
    mutate {
      uppercase => [ "level" ]
    }
  }
  
  # 불필요한 필드 제거
  mutate {
    remove_field => [ "agent", "ecs", "log", "input", "host" ]
  }
}

# 출력 설정
output {
  # Elasticsearch로 데이터 전송
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    template_name => "logs"
    template => {
      "index_patterns" => ["logs-*"],
      "settings" => {
        "number_of_shards" => 1,
        "number_of_replicas" => 0
      },
      "mappings" => {
        "properties" => {
          "@timestamp" => { "type" => "date" },
          "level" => { "type" => "keyword" },
          "message" => { "type" => "text" },
          "service" => { "type" => "keyword" }
        }
      }
    }
  }
  
  # 콘솔에 디버그 출력
  stdout {
    codec => rubydebug
  }
}
```

### 샘플 로그 파일 생성

#### logs/application.log
```
2024-01-15 10:00:01 [INFO] Application started successfully service=user-api version=1.2.3
2024-01-15 10:00:15 [DEBUG] Database connection established host=db-server port=5432
2024-01-15 10:01:23 [WARN] High memory usage detected memory_usage=85% threshold=80%
2024-01-15 10:02:45 [ERROR] Failed to authenticate user user_id=12345 reason=invalid_token
2024-01-15 10:03:10 [INFO] User login successful user_id=67890 ip=192.168.1.100
```

#### logs/nginx.log
```json
{"timestamp":"2024-01-15T10:00:01.000Z","level":"INFO","service":"nginx","message":"GET /api/users/12345 200 125ms","ip":"192.168.1.100","method":"GET","url":"/api/users/12345","status":200,"response_time":125}
{"timestamp":"2024-01-15T10:00:15.000Z","level":"INFO","service":"nginx","message":"POST /api/auth/login 200 45ms","ip":"192.168.1.101","method":"POST","url":"/api/auth/login","status":200,"response_time":45}
{"timestamp":"2024-01-15T10:01:23.000Z","level":"WARN","service":"nginx","message":"GET /api/data 429 12ms","ip":"192.168.1.102","method":"GET","url":"/api/data","status":429,"response_time":12}
{"timestamp":"2024-01-15T10:02:45.000Z","level":"ERROR","service":"nginx","message":"POST /api/upload 500 5000ms","ip":"192.168.1.103","method":"POST","url":"/api/upload","status":500,"response_time":5000}
```

## ELK 스택 실행 및 검증

### 컨테이너 실행

#### 스택 시작
```bash
# 백그라운드에서 모든 서비스 시작
docker-compose up -d

# 서비스 상태 확인
docker-compose ps

# 로그 확인
docker-compose logs -f
```

#### 개별 서비스 로그 확인
```bash
# Elasticsearch 로그
docker-compose logs elasticsearch

# Logstash 로그  
docker-compose logs logstash

# Kibana 로그
docker-compose logs kibana
```

### 서비스 상태 확인

#### Elasticsearch 상태 확인
```bash
# 클러스터 상태
curl -X GET "localhost:9200/_cluster/health?pretty"

# 노드 정보
curl -X GET "localhost:9200/_nodes?pretty"

# 인덱스 목록
curl -X GET "localhost:9200/_cat/indices?v"
```

**정상 응답 예시**:
```json
{
  "cluster_name" : "elk-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 1,
  "active_shards" : 1,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0
}
```

#### Logstash 상태 확인
```bash
# 파이프라인 상태
curl -X GET "localhost:9600/_node/stats/pipelines?pretty"

# 플러그인 정보
curl -X GET "localhost:9600/_node/plugins?pretty"
```

#### Kibana 접속 확인
```bash
# 브라우저에서 접속
open http://localhost:5601

# 또는 curl로 상태 확인
curl -X GET "localhost:5601/api/status"
```

## Kibana 초기 설정

### 인덱스 패턴 생성

#### 단계별 설정
1. **Kibana 접속**: http://localhost:5601
2. **Management 메뉴**: 좌측 메뉴에서 Stack Management 선택
3. **Index Patterns**: Kibana > Index Patterns 선택
4. **Create index pattern**: 버튼 클릭
5. **인덱스 패턴 입력**: `logs-*` 입력
6. **Time field**: `@timestamp` 선택
7. **Create index pattern**: 완료

### 데이터 확인

#### Discover 메뉴 사용
1. **Discover 선택**: 좌측 메뉴에서 Discover 클릭
2. **인덱스 패턴 선택**: logs-* 패턴 선택
3. **시간 범위 설정**: 우상단에서 시간 범위 조정
4. **데이터 탐색**: 로그 데이터 확인 및 필터링

#### 기본 검색 쿼리 예제
```
# 특정 로그 레벨 검색
level: ERROR

# 서비스별 필터링
service: nginx

# 시간 범위와 조건 조합
level: ERROR AND @timestamp >= now-1h

# 메시지 내용 검색
message: "authentication failed"

# 숫자 필드 범위 검색
response_time: [100 TO 1000]
```

## 간단한 로그 수집 워크플로우

### 실시간 로그 생성 스크립트

#### log-generator.sh
```bash
#!/bin/bash

LOG_FILE="./logs/realtime.log"
SERVICES=("user-api" "payment-api" "notification-service")
LEVELS=("INFO" "WARN" "ERROR" "DEBUG")

while true; do
  # 랜덤 서비스 선택
  SERVICE=${SERVICES[$RANDOM % ${#SERVICES[@]}]}
  
  # 랜덤 로그 레벨 선택
  LEVEL=${LEVELS[$RANDOM % ${#LEVELS[@]}]}
  
  # 현재 시간 생성
  TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
  
  # 랜덤 메시지 생성
  case $LEVEL in
    "ERROR")
      MESSAGE="Database connection failed host=db-server error=timeout"
      ;;
    "WARN") 
      MESSAGE="High CPU usage detected cpu_usage=90% threshold=80%"
      ;;
    "INFO")
      MESSAGE="Request processed successfully user_id=$RANDOM response_time=${RANDOM:0:3}ms"
      ;;
    "DEBUG")
      MESSAGE="Cache hit ratio cache_hits=$RANDOM total_requests=$RANDOM"
      ;;
  esac
  
  # 로그 파일에 기록
  echo "$TIMESTAMP [$LEVEL] $MESSAGE service=$SERVICE" >> $LOG_FILE
  
  # 1-5초 대기
  sleep $((RANDOM % 5 + 1))
done
```

#### 스크립트 실행
```bash
# 실행 권한 부여
chmod +x log-generator.sh

# 백그라운드에서 실행
./log-generator.sh &

# PID 확인
echo $! > log-generator.pid
```

### 로그 확인 및 모니터링

#### 실시간 로그 확인
```bash
# 로그 파일 실시간 모니터링
tail -f logs/realtime.log

# Logstash 처리 상태 확인
docker-compose logs -f logstash

# Elasticsearch 인덱스 문서 수 확인
curl -X GET "localhost:9200/logs-*/_count?pretty"
```

## 문제 해결

### 일반적인 문제들

#### 1. Elasticsearch 시작 실패
**증상**: 컨테이너가 반복적으로 재시작됨

**해결방법**:
```bash
# 로그 확인
docker-compose logs elasticsearch

# 메모리 부족 시 힙 사이즈 조정
# docker-compose.yml에서 ES_JAVA_OPTS 수정
ES_JAVA_OPTS=-Xms512m -Xmx512m

# 권한 문제 시 데이터 디렉터리 권한 수정
sudo chown -R 1000:1000 elasticsearch/data
```

#### 2. Logstash 파이프라인 오류
**증상**: 로그가 Elasticsearch에 저장되지 않음

**해결방법**:
```bash
# 설정 파일 문법 검사
docker exec logstash /usr/share/logstash/bin/logstash \
  -f /usr/share/logstash/pipeline/logstash.conf --config.test_and_exit

# 파이프라인 상태 확인
curl -X GET "localhost:9600/_node/stats/pipelines?pretty"

# 로그 파일 권한 확인
ls -la logs/
```

#### 3. Kibana 접속 불가
**증상**: http://localhost:5601 접속 시 오류

**해결방법**:
```bash
# Kibana 서비스 상태 확인
docker-compose ps kibana

# Elasticsearch 연결 확인
curl -X GET "localhost:9200/_cluster/health"

# Kibana 로그 확인
docker-compose logs kibana
```

### 성능 최적화

#### 메모리 설정 조정
```yaml
# docker-compose.yml에서 힙 메모리 조정
elasticsearch:
  environment:
    - "ES_JAVA_OPTS=-Xms2g -Xmx2g"  # 시스템 RAM의 50% 할당

logstash:
  environment:
    - "LS_JAVA_OPTS=-Xms1g -Xmx1g"  # Logstash 메모리 증가
```

#### 디스크 I/O 최적화
```bash
# SSD 사용 시 mount 옵션 최적화
mount -o noatime,nodiratime /dev/ssd /var/lib/docker

# 로그 파일 로테이션 설정
docker-compose logs --tail=1000 > limited-logs.txt
```

## 다음 단계

Phase 1 완료 후 다음 학습 목표:
- **Beats 제품군 통합**: Filebeat, Metricbeat 설정
- **고급 Logstash 필터링**: Grok 패턴 및 데이터 변환
- **Elasticsearch 인덱스 최적화**: 매핑 및 성능 튜닝

---

이 문서를 통해 Docker 기반 ELK 스택 환경을 성공적으로 구축했다면, 실제 로그 데이터 처리 및 분석을 위한 다음 단계로 진행할 준비가 완료되었습니다.