# 2.2 Logstash 고급 구성

## Overview
Logstash는 강력한 데이터 파이프라인 도구로, 다양한 소스에서 데이터를 수집하고 변환하여 Elasticsearch로 전송합니다. 이 문서에서는 고급 파이프라인 구성과 최적화 기법을 다룹니다.

## Logstash 파이프라인 심화

### Input 플러그인 상세

#### Beats Input
```ruby
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate_authorities => ["/path/to/ca.crt"]
    ssl_certificate => "/path/to/server.crt"
    ssl_key => "/path/to/server.key"
    ssl_verify_mode => "force_peer"
  }
}
```

#### File Input
```ruby
input {
  file {
    path => ["/var/log/apache2/access.log", "/var/log/apache2/error.log"]
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601}"
      negate => true
      what => "previous"
    }
    tags => ["apache", "webserver"]
  }
}
```

#### Syslog Input
```ruby
input {
  syslog {
    port => 514
    type => "syslog"
    facility_labels => ["kernel", "user", "mail", "daemon"]
    severity_labels => ["emergency", "alert", "critical", "error"]
  }
}
```

#### HTTP Input
```ruby
input {
  http {
    port => 8080
    user => "logstash"
    password => "password"
    ssl => true
    ssl_certificate => "/path/to/cert.pem"
    ssl_key => "/path/to/key.pem"
    codec => json
  }
}
```

### Filter 플러그인 고급 활용

#### Grok 패턴 최적화
```ruby
filter {
  # 커스텀 패턴 정의
  grok {
    patterns_dir => ["/etc/logstash/patterns"]
    match => { 
      "message" => "%{CUSTOM_APACHE_LOG}" 
    }
    tag_on_failure => ["_grokparsefailure"]
  }
  
  # 다중 패턴 매칭
  grok {
    match => {
      "message" => [
        "%{APACHE_ACCESS_LOG}",
        "%{APACHE_ERROR_LOG}",
        "%{NGINX_ACCESS_LOG}"
      ]
    }
  }
}
```

#### Mutate를 통한 데이터 변환
```ruby
filter {
  mutate {
    # 필드 이름 변경
    rename => { "host" => "hostname" }
    
    # 필드 타입 변환
    convert => { 
      "response_time" => "float"
      "status_code" => "integer"
    }
    
    # 필드 추가
    add_field => { 
      "environment" => "production"
      "[@metadata][index]" => "logs-%{+YYYY.MM.dd}"
    }
    
    # 필드 제거
    remove_field => ["@version", "path", "host"]
    
    # 문자열 처리
    gsub => [
      "message", "\t", " ",
      "user_agent", '"', ""
    ]
  }
}
```

#### Date 플러그인 고급 사용
```ruby
filter {
  date {
    match => [ 
      "timestamp", 
      "dd/MMM/yyyy:HH:mm:ss Z",
      "ISO8601",
      "UNIX"
    ]
    target => "@timestamp"
    timezone => "Asia/Seoul"
    locale => "en"
  }
}
```

#### GeoIP 및 위치 정보 추가
```ruby
filter {
  geoip {
    source => "client_ip"
    target => "geoip"
    database => "/etc/logstash/GeoLite2-City.mmdb"
    add_field => { 
      "[geoip][coordinates]" => "%{[geoip][longitude]}"
      "[geoip][coordinates]" => "%{[geoip][latitude]}"
    }
  }
}
```

### Output 플러그인 최적화

#### Elasticsearch Output 고급 설정
```ruby
output {
  elasticsearch {
    hosts => ["es-node-1:9200", "es-node-2:9200", "es-node-3:9200"]
    index => "%{[@metadata][index]}"
    
    # 템플릿 관리
    template_name => "logstash"
    template => "/etc/logstash/templates/logstash.json"
    template_overwrite => true
    
    # 성능 최적화
    workers => 4
    flush_size => 1000
    idle_flush_time => 1
    
    # 재시도 설정
    retry_on_conflict => 3
    retry_max_interval => 5
    
    # 보안 설정
    user => "logstash_writer"
    password => "${LOGSTASH_PASSWORD}"
    ssl => true
    cacert => "/etc/ssl/certs/ca.crt"
  }
}
```

#### 조건부 출력
```ruby
output {
  if [type] == "apache" {
    elasticsearch {
      hosts => ["localhost:9200"]
      index => "apache-%{+YYYY.MM.dd}"
    }
  } else if [type] == "nginx" {
    elasticsearch {
      hosts => ["localhost:9200"]
      index => "nginx-%{+YYYY.MM.dd}"
    }
  } else {
    elasticsearch {
      hosts => ["localhost:9200"]
      index => "misc-%{+YYYY.MM.dd}"
    }
  }
}
```

## 데이터 변환 기법

### 복합 필터링 파이프라인
```ruby
filter {
  # 1단계: 기본 파싱
  if [type] == "apache_access" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }
  
  # 2단계: 데이터 보강
  if [clientip] {
    geoip {
      source => "clientip"
      target => "geoip"
    }
  }
  
  # 3단계: 사용자 에이전트 파싱
  if [agent] {
    useragent {
      source => "agent"
      target => "user_agent"
    }
  }
  
  # 4단계: 응답 시간 분류
  if [response] {
    if [response] >= 500 {
      mutate { add_tag => ["error"] }
    } else if [response] >= 400 {
      mutate { add_tag => ["client_error"] }
    } else if [response] >= 300 {
      mutate { add_tag => ["redirect"] }
    } else if [response] >= 200 {
      mutate { add_tag => ["success"] }
    }
  }
}
```

### 정규표현식 고급 활용
```ruby
filter {
  # SQL 쿼리 파싱
  if [message] =~ /SELECT|INSERT|UPDATE|DELETE/ {
    grok {
      match => { 
        "message" => "(?<query_type>SELECT|INSERT|UPDATE|DELETE).*?FROM\s+(?<table>\w+)" 
      }
    }
  }
  
  # 이메일 주소 추출
  if [message] =~ /@/ {
    grok {
      match => { 
        "message" => "(?<email>[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})" 
      }
    }
  }
}
```

### Aggregate 플러그인 활용
```ruby
filter {
  aggregate {
    task_id => "%{session_id}"
    code => "
      map['total_requests'] ||= 0
      map['total_requests'] += 1
      map['total_bytes'] ||= 0
      map['total_bytes'] += event.get('bytes').to_i
    "
    map_action => "update"
    timeout => 300
  }
}
```

## 조건부 처리 및 라우팅

### 태그 기반 라우팅
```ruby
filter {
  if "error" in [tags] {
    mutate {
      add_field => { "priority" => "high" }
      add_field => { "alert_team" => "ops" }
    }
  }
  
  if [log_level] == "DEBUG" and [environment] == "production" {
    drop { }
  }
}
```

### 메타데이터 활용
```ruby
filter {
  mutate {
    add_field => { 
      "[@metadata][index_prefix]" => "logs"
      "[@metadata][document_type]" => "%{[type]}"
    }
  }
}

output {
  elasticsearch {
    index => "%{[@metadata][index_prefix]}-%{[@metadata][document_type]}-%{+YYYY.MM.dd}"
  }
}
```

## 오류 처리 및 데드 레터 큐

### 파싱 실패 처리
```ruby
filter {
  grok {
    match => { "message" => "%{APACHE_ACCESS_LOG}" }
    tag_on_failure => ["_grokparsefailure"]
  }
  
  if "_grokparsefailure" in [tags] {
    mutate {
      add_field => { "parse_error" => "true" }
      add_field => { "original_message" => "%{message}" }
    }
  }
}
```

### Dead Letter Queue 활용
```yaml
# logstash.yml
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 1024mb
```

```ruby
input {
  dead_letter_queue {
    path => "/var/lib/logstash/dead_letter_queue"
    commit_offsets => true
  }
}
```

## 파이프라인 최적화 및 성능 튜닝

### 메모리 관리
```yaml
# jvm.options
-Xms2g
-Xmx2g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

### 파이프라인 설정
```yaml
# pipelines.yml
- pipeline.id: main
  path.config: "/etc/logstash/conf.d/*.conf"
  pipeline.workers: 4
  pipeline.batch.size: 1000
  pipeline.batch.delay: 50
  queue.type: persisted
  queue.max_bytes: 1gb
```

### 멀티 파이프라인 구성
```yaml
# pipelines.yml
- pipeline.id: apache_logs
  path.config: "/etc/logstash/conf.d/apache.conf"
  pipeline.workers: 2
  
- pipeline.id: application_logs  
  path.config: "/etc/logstash/conf.d/app.conf"
  pipeline.workers: 4
  
- pipeline.id: security_logs
  path.config: "/etc/logstash/conf.d/security.conf"
  pipeline.workers: 1
```

## Best Practices

### 성능 최적화
- **파이프라인 워커 수**: CPU 코어 수와 동일하게 설정
- **배치 크기**: 메모리와 처리량 균형 고려
- **Persistent Queue**: 데이터 손실 방지를 위한 필수 설정
- **JVM 튜닝**: 힙 크기와 GC 알고리즘 최적화

### 모니터링
- **파이프라인 메트릭**: 처리량, 지연 시간, 에러율 모니터링
- **리소스 사용량**: CPU, 메모리, 디스크 I/O 추적
- **큐 상태**: 입력/출력 큐 크기 모니터링

## Benefits and Challenges

### Benefits
- **유연성**: 다양한 데이터 소스와 변환 규칙 지원
- **확장성**: 멀티 파이프라인과 워커를 통한 수평 확장
- **신뢰성**: Persistent Queue와 재시도 메커니즘
- **플러그인 생태계**: 풍부한 입력/필터/출력 플러그인

### Challenges
- **설정 복잡성**: 복잡한 파이프라인 구성 및 디버깅
- **메모리 사용량**: 대량 데이터 처리 시 높은 메모리 요구
- **성능 튜닝**: 워크로드에 맞는 최적화 필요
- **버전 관리**: 플러그인 및 Elastic Stack 버전 호환성