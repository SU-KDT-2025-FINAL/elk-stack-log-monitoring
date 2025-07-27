# 2.1 Beats 제품군 통합

## Overview
Beats는 Elastic Stack의 경량 데이터 수집기로, 다양한 소스에서 데이터를 수집하여 Elasticsearch나 Logstash로 전송합니다. 이 문서에서는 각 Beats 제품의 특징과 설정 방법을 다룹니다.

## Beats 제품군 개요

### Filebeat - 로그 파일 수집
- **목적**: 로그 파일 모니터링 및 전송
- **주요 기능**:
  - 다중 로그 파일 동시 모니터링
  - 파일 회전(rotation) 자동 감지
  - 네트워크 장애 시 백프레셔 처리
  - 중복 전송 방지 및 최소 한 번 전송 보장

### Metricbeat - 시스템 메트릭 수집
- **목적**: 시스템 및 서비스 메트릭 수집
- **지원 메트릭**:
  - 시스템 메트릭: CPU, 메모리, 디스크, 네트워크
  - 서비스 메트릭: Apache, Nginx, MySQL, Redis 등
  - 컨테이너 메트릭: Docker, Kubernetes

### Packetbeat - 네트워크 트래픽 분석
- **목적**: 실시간 네트워크 패킷 분석
- **지원 프로토콜**:
  - HTTP/HTTPS 트랜잭션
  - MySQL, PostgreSQL 쿼리
  - DNS 요청/응답
  - Redis 명령어

### Winlogbeat - Windows 이벤트 로그
- **목적**: Windows 이벤트 로그 수집
- **지원 로그**:
  - Security 이벤트
  - Application 로그
  - System 로그
  - Custom 이벤트 로그

## Beats 설정 및 구성

### Filebeat 기본 설정

```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
    - /var/log/apache2/*
  fields:
    logtype: application
  fields_under_root: true

- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  fields:
    logtype: webserver
  multiline.pattern: '^\d{4}-\d{2}-\d{2}'
  multiline.negate: true
  multiline.match: after

output.elasticsearch:
  hosts: ["localhost:9200"]
  index: "filebeat-%{+yyyy.MM.dd}"

processors:
- add_host_metadata:
    when.not.contains.tags: forwarded
```

### Metricbeat 설정

```yaml
# metricbeat.yml
metricbeat.modules:
- module: system
  metricsets:
    - cpu
    - memory
    - network
    - process
    - filesystem
  enabled: true
  period: 10s
  processes: ['.*']

- module: apache
  metricsets: ["status"]
  period: 10s
  hosts: ["http://127.0.0.1/server-status?auto"]

- module: docker
  metricsets:
    - container
    - cpu
    - memory
    - network
  hosts: ["unix:///var/run/docker.sock"]
  period: 10s

output.elasticsearch:
  hosts: ["localhost:9200"]
  index: "metricbeat-%{+yyyy.MM.dd}"
```

### Packetbeat 설정

```yaml
# packetbeat.yml
packetbeat.interfaces.device: any

packetbeat.protocols:
- type: http
  ports: [80, 8080, 8000, 5000, 8002]
  real_ip_header: "X-Forwarded-For"

- type: mysql
  ports: [3306]

- type: redis
  ports: [6379]

output.elasticsearch:
  hosts: ["localhost:9200"]
  index: "packetbeat-%{+yyyy.MM.dd}"

processors:
- include_fields:
    fields: ["@timestamp", "type", "status", "responsetime"]
```

## 다양한 데이터 소스 연결

### 로그 파일 수집 전략

```yaml
# 애플리케이션별 로그 분리
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/app1/*.log
  fields:
    service: app1
    environment: production
  
- type: log
  enabled: true
  paths:
    - /var/log/app2/*.log
  fields:
    service: app2
    environment: production
```

### Docker 컨테이너 로그 수집

```yaml
# Docker 컨테이너 로그
filebeat.inputs:
- type: container
  paths:
    - '/var/lib/docker/containers/*/*.log'
  processors:
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"
```

### Kubernetes 환경 설정

```yaml
# Kubernetes DaemonSet으로 배포
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |
    filebeat.autodiscover:
      providers:
        - type: kubernetes
          node: ${NODE_NAME}
          hints.enabled: true
    
    output.elasticsearch:
      hosts: ["elasticsearch:9200"]
```

## 데이터 필터링 및 전처리

### Processors 활용

```yaml
# 공통 processors 설정
processors:
- add_host_metadata:
    when.not.contains.tags: forwarded

- add_cloud_metadata: ~

- add_docker_metadata: ~

- add_kubernetes_metadata: ~

- drop_fields:
    fields: ["agent", "ecs", "host.architecture"]

- timestamp:
    field: timestamp
    layouts:
      - '2006-01-02T15:04:05.000Z'
      - '2006-01-02T15:04:05Z'
    test:
      - '2020-12-07T12:34:56.123Z'
```

### 조건부 필터링

```yaml
# 조건부 데이터 처리
processors:
- drop_event:
    when:
      regexp:
        message: 'DEBUG|TRACE'

- script:
    lang: javascript
    source: >
      function process(event) {
        if (event.Get("log.level") === "ERROR") {
          event.Put("priority", "high");
        }
      }
```

## Best Practices

### 성능 최적화
- **배치 크기 조정**: `output.elasticsearch.bulk_max_size` 설정
- **압축 활용**: `output.elasticsearch.compression_level` 설정
- **큐 설정**: `queue.mem.events` 및 `queue.mem.flush.min_events` 조정
- **리소스 제한**: CPU 및 메모리 사용량 모니터링

### 보안 설정
- **TLS 암호화**: Elasticsearch 연결 시 SSL/TLS 사용
- **인증 구성**: API 키 또는 사용자명/비밀번호 설정
- **인덱스 템플릿**: 적절한 매핑 및 설정 미리 정의

### 모니터링 및 운영
- **라이브러리 로그**: Beats 자체 로그 모니터링
- **메트릭 수집**: Beats 성능 메트릭 Elasticsearch 전송
- **상태 확인**: HTTP 엔드포인트를 통한 헬스 체크

## Benefits and Challenges

### Benefits
- **경량성**: 최소한의 리소스로 효율적 데이터 수집
- **확장성**: 수평 확장을 통한 대규모 환경 지원
- **신뢰성**: 장애 복구 및 재전송 메커니즘
- **다양성**: 다양한 데이터 소스 및 프로토콜 지원

### Challenges
- **설정 복잡성**: 대규모 환경에서의 설정 관리
- **네트워크 대역폭**: 대량 데이터 전송 시 네트워크 부하
- **버전 호환성**: Elastic Stack 버전 간 호환성 관리
- **모니터링**: Beats 자체 상태 모니터링 필요