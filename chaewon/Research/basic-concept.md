# ELK Stack 로그 모니터링 기본 개념 가이드

## 1. ELK Stack 개요

ELK Stack은 **Elasticsearch**, **Logstash**, **Kibana**의 세 가지 오픈소스 프로젝트의 조합으로, 로그 데이터를 수집, 저장, 검색, 분석 및 시각화하는 통합 플랫폼입니다. 현재는 **Beats**가 추가되어 **Elastic Stack**이라고도 불립니다.

### 주요 장점
- 실시간 로그 분석 및 모니터링
- 확장 가능한 분산 아키텍처
- 강력한 검색 및 필터링 기능
- 직관적인 데이터 시각화
- 오픈소스 기반의 비용 효율성

## 2. ELK Stack 구성 요소

### 2.1 Elasticsearch
**분산 검색 및 분석 엔진**

- **역할**: 로그 데이터의 저장, 인덱싱, 검색
- **특징**:
    - Apache Lucene 기반
    - RESTful API 제공
    - 실시간 검색 지원
    - 수평적 확장 가능
    - JSON 문서 기반 저장

- **핵심 개념**:
    - **Index**: 데이터베이스와 유사한 개념
    - **Document**: 인덱스 내의 개별 데이터 단위
    - **Mapping**: 문서의 필드 타입 정의
    - **Shard**: 인덱스를 분할한 단위
    - **Replica**: 샤드의 복제본

### 2.2 Logstash
**데이터 수집 및 처리 파이프라인**

- **역할**: 다양한 소스에서 로그 수집, 변환, Elasticsearch로 전송
- **특징**:
    - 200+ 플러그인 지원
    - 실시간 데이터 파이프라인
    - 데이터 파싱 및 변환
    - 필터링 및 강화 기능

- **파이프라인 구조**:
  ```
  Input → Filter → Output
  ```
    - **Input**: 데이터 소스 (파일, 네트워크, 데이터베이스 등)
    - **Filter**: 데이터 파싱, 변환, 강화
    - **Output**: 목적지 (Elasticsearch, 파일, 다른 시스템)

### 2.3 Kibana
**데이터 시각화 및 대시보드**

- **역할**: Elasticsearch 데이터의 시각화 및 분석 인터페이스
- **특징**:
    - 웹 기반 인터페이스
    - 실시간 대시보드
    - 다양한 차트 및 그래프
    - 알림 및 모니터링 기능

- **주요 기능**:
    - **Discover**: 로그 데이터 탐색 및 검색
    - **Visualize**: 차트 및 그래프 생성
    - **Dashboard**: 여러 시각화를 조합한 대시보드
    - **Alerting**: 임계값 기반 알림

### 2.4 Beats
**경량 데이터 수집기**

- **역할**: 다양한 시스템에서 데이터를 수집하여 Elasticsearch 또는 Logstash로 전송
- **주요 유형**:
    - **Filebeat**: 로그 파일 수집
    - **Metricbeat**: 시스템 및 서비스 메트릭 수집
    - **Packetbeat**: 네트워크 패킷 분석
    - **Winlogbeat**: Windows 이벤트 로그 수집
    - **Heartbeat**: 서비스 가용성 모니터링

## 3. 로그 모니터링 아키텍처

### 3.1 기본 아키텍처
```
애플리케이션/시스템 → Beats/Logstash → Elasticsearch → Kibana
```

### 3.2 고급 아키텍처
```
다중 소스 → Beats → Logstash → Elasticsearch 클러스터 → Kibana
           ↓
        Message Queue (Redis/Kafka)
```

### 3.3 구성 요소별 역할
- **데이터 소스**: 애플리케이션, 서버, 네트워크 장비
- **수집 계층**: Beats, Logstash
- **저장 계층**: Elasticsearch 클러스터
- **분석 계층**: Kibana 대시보드
- **알림 계층**: Watcher, ElastAlert

## 4. 주요 사용 사례

### 4.1 애플리케이션 로그 모니터링
- 에러 로그 추적 및 분석
- 성능 병목 지점 식별
- 사용자 행동 패턴 분석
- API 호출 모니터링

### 4.2 인프라스트럭처 모니터링
- 시스템 리소스 사용률 추적
- 서버 상태 모니터링
- 네트워크 트래픽 분석
- 보안 이벤트 감지

### 4.3 비즈니스 인텔리전스
- 실시간 비즈니스 메트릭
- 고객 행동 분석
- 매출 및 전환율 추적
- A/B 테스트 결과 분석

## 5. 로그 데이터 구조 및 형식

### 5.1 구조화된 로그
```json
{
  "@timestamp": "2024-01-15T10:30:00.000Z",
  "level": "ERROR",
  "message": "Database connection failed",
  "service": "user-service",
  "host": "web-server-01",
  "thread": "main",
  "exception": "SQLException: Connection timeout"
}
```

### 5.2 일반적인 로그 필드
- **@timestamp**: 로그 발생 시간
- **level**: 로그 레벨 (ERROR, WARN, INFO, DEBUG)
- **message**: 로그 메시지
- **service**: 서비스 이름
- **host**: 호스트 정보
- **user_id**: 사용자 식별자
- **request_id**: 요청 추적 ID

## 6. 검색 및 쿼리

### 6.1 Elasticsearch Query DSL
```json
{
  "query": {
    "bool": {
      "must": [
        {"match": {"level": "ERROR"}},
        {"range": {"@timestamp": {"gte": "now-1h"}}}
      ]
    }
  }
}
```

### 6.2 Kibana Query Language (KQL)
```
level:ERROR AND @timestamp:[now-1h TO now]
service:"user-service" AND message:*timeout*
```

### 6.3 일반적인 검색 패턴
- 시간 범위 기반 검색
- 로그 레벨별 필터링
- 서비스별 로그 분석
- 정규식 패턴 매칭
- 집계 및 통계 쿼리

## 7. 성능 최적화 고려사항

### 7.1 Elasticsearch 최적화
- **인덱스 설계**: 시간 기반 인덱스 패턴 사용
- **샤드 크기**: 10-50GB 권장
- **매핑 최적화**: 불필요한 필드 저장 제외
- **리플리카 설정**: 가용성과 성능 균형

### 7.2 Logstash 최적화
- **파이프라인 튜닝**: 워커 수 및 배치 크기 조정
- **필터 최적화**: 불필요한 파싱 제거
- **메모리 관리**: JVM 힙 크기 설정
- **큐 설정**: 영구 큐 사용 고려

### 7.3 Beats 최적화
- **수집 주기**: 시스템 부하와 실시간성 균형
- **버퍼 크기**: 네트워크 상황에 맞는 설정
- **필드 선택**: 필요한 필드만 수집

## 8. 보안 고려사항

### 8.1 접근 제어
- **Authentication**: 사용자 인증
- **Authorization**: 역할 기반 접근 제어
- **Network Security**: TLS/SSL 암호화
- **API Key**: 서비스 간 인증

### 8.2 데이터 보호
- **Index Level Security**: 인덱스별 접근 권한
- **Field Level Security**: 민감한 필드 보호
- **Document Level Security**: 문서별 필터링
- **Audit Logging**: 접근 기록 추적

## 9. 모니터링 및 알림

### 9.1 핵심 메트릭
- **시스템 메트릭**: CPU, 메모리, 디스크 사용률
- **Elasticsearch 메트릭**: 인덱싱 속도, 검색 성능
- **애플리케이션 메트릭**: 에러율, 응답 시간
- **비즈니스 메트릭**: 사용자 활동, 전환율

### 9.2 알림 설정
- **임계값 기반**: 메트릭이 특정 값 초과 시
- **이상 탐지**: 패턴 변화 감지
- **로그 기반**: 특정 로그 패턴 발생 시
- **복합 조건**: 여러 조건 조합

## 10. 베스트 프랙티스

### 10.1 로그 설계
- 구조화된 로그 형식 사용 (JSON)
- 일관된 필드 명명 규칙
- 적절한 로그 레벨 사용
- 상관 관계 ID 포함

### 10.2 운영 관리
- 정기적인 인덱스 관리 (ILM 정책)
- 백업 및 복구 계획
- 모니터링 대시보드 구성
- 용량 계획 및 확장 전략

### 10.3 개발 프로세스
- 로그 표준화 가이드라인
- 개발 환경에서의 테스트
- 성능 영향 최소화
- 문서화 및 교육
