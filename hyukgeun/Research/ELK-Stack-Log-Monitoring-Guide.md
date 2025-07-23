# ELK 스택(Elasticsearch, Logstash, Kibana)을 활용한 로그 수집 및 시각화 시스템 구축 가이드

## 1. ELK 스택 개요

- **Elasticsearch**: 분산형 검색 및 분석 엔진. 대용량 로그 데이터 저장 및 검색에 최적화.
- **Logstash**: 다양한 소스에서 로그를 수집, 변환, Elasticsearch로 전달하는 데이터 파이프라인 도구.
- **Kibana**: Elasticsearch 데이터를 시각화하고 대시보드를 제공하는 웹 UI.
- **Beats**: Filebeat, Metricbeat 등 경량 데이터 수집기. 다양한 로그/메트릭 자동 수집에 활용

### 1-1. 아키텍처 심화

- **분산 구조**: ELK 스택은 단일 서버뿐 아니라 다수의 서버(노드)로 구성된 클러스터 환경을 지원합니다. 대용량 데이터 처리, 장애 복구, 확장성 확보를 위해 분산 아키텍처가 필수적입니다.
    - Elasticsearch는 여러 노드로 클러스터를 구성하며, 각 노드는 역할(마스터, 데이터, 인제스트, 코디네이터 등)을 분담합니다.
    - Logstash와 Beats 역시 여러 인스턴스를 분산 배치하여 수집 지연 및 장애에 대응할 수 있습니다.

- **노드 역할**:
    - **마스터 노드**: 클러스터 상태 관리, 인덱스 생성/삭제, 노드 관리 등 메타데이터 담당. 고가용성을 위해 3개 이상 권장.
    - **데이터 노드**: 실제 데이터 저장 및 검색, 인덱싱 작업 담당. 스토리지와 I/O 성능이 중요.
    - **인제스트 노드**: 파이프라인 기반 데이터 전처리(파싱, 변환 등) 담당.
    - **코디네이터 노드**: 클라이언트 요청을 받아 적절한 노드로 분배.

- **샤드(Shard)와 레플리카(Replica)**:
    - **샤드**: 인덱스를 여러 조각으로 분할하여 분산 저장. 대용량 데이터의 병렬 처리와 확장성 확보.
    - **레플리카**: 샤드의 복제본. 장애 발생 시 데이터 유실 방지 및 읽기 성능 향상.
    - 샤드/레플리카 개수는 인덱스 생성 시 설계에 따라 결정하며, 운영 중에도 조정 가능.

- **데이터 흐름**:
    1. **로그 소스**(서버, 애플리케이션, 네트워크 등)에서 로그 발생
    2. **Beats**(Filebeat 등)가 로그를 수집하여 Logstash 또는 Elasticsearch로 전송
    3. **Logstash**가 로그를 파싱, 변환, 정제하여 Elasticsearch에 저장
    4. **Elasticsearch**는 데이터를 색인(indexing)하여 검색/분석이 가능하도록 저장
    5. **Kibana**가 Elasticsearch 데이터를 시각화, 대시보드, 알림 등으로 제공

- **확장성 및 고가용성**:
    - 노드 추가를 통한 수평 확장(Scale-out)
    - 샤드/레플리카 조정으로 데이터 분산 및 장애 복구
    - 각 컴포넌트의 이중화 및 로드밸런싱

- **실전 팁**:
    - 클러스터 설계 시 마스터 노드와 데이터 노드를 분리하여 안정성 확보
    - 샤드 개수는 데이터량과 노드 수를 고려해 설계(너무 많거나 적으면 성능 저하)
    - 인덱스 롤오버, ILM(수명주기 관리)로 장기 운영 시 관리 자동화
    - 각 노드의 리소스(CPU, 메모리, 디스크) 모니터링 필수

### 1-2. 운영 환경별 구축 전략

- **온프레미스(자체 서버) 환경**:
    - 하드웨어 스펙(메모리, 디스크 IOPS, 네트워크 대역폭) 사전 산정 필수
    - 장애 복구를 위한 이중화(노드, 네트워크, 전원 등) 설계
    - 보안: 내부망 분리, 방화벽, 접근제어, 인증서 기반 통신 적용
    - 스토리지: SSD 권장, RAID 구성, 스냅샷 백업 자동화

- **클라우드 환경(AWS, GCP, Azure 등)**:
    - Managed Elasticsearch 서비스(AWS OpenSearch, Elastic Cloud 등) 활용 가능
    - 오토스케일링, 스냅샷 백업, 모니터링 등 클라우드 네이티브 기능 적극 활용
    - 네트워크: VPC, 보안그룹, 프라이빗 엔드포인트 구성
    - 비용 최적화: 데이터 수명주기 관리(ILM), 저비용 스토리지 계층 활용

- **하이브리드/멀티클러스터 환경**:
    - 여러 지역/데이터센터에 클러스터 분산 배치
    - Cross-cluster search, replication 등으로 데이터 통합/분산 운영
    - 각 환경별 보안 정책 및 데이터 동기화 전략 수립

### 1-3. ELK 스택의 주요 활용 사례

- **애플리케이션/시스템 로그 통합 모니터링**:
    - 다양한 서버, 컨테이너, 클라우드 리소스의 로그를 통합 수집 및 분석
    - 장애 탐지, 성능 이슈, 보안 이벤트 실시간 탐지

- **보안 관제(SIEM) 및 이상 징후 탐지**:
    - Elastic SIEM, Watcher, Alerting을 활용한 실시간 보안 이벤트 탐지 및 대응
    - 외부 위협 인텔리전스 연동, 자동화된 알림 및 대응

- **비즈니스 데이터 분석 및 대시보드**:
    - 서비스 이용 패턴, 트랜잭션 분석, 사용자 행동 분석 등 비즈니스 인사이트 도출
    - 실시간 KPI 대시보드, 경영진 보고용 시각화

- **DevOps/CI-CD 파이프라인 모니터링**:
    - 배포, 빌드, 테스트 로그 실시간 집계 및 장애 알림
    - 인프라 자동화와 연계한 운영 효율화

- **IoT/센서 데이터 실시간 분석**:
    - 대규모 IoT 디바이스의 로그/이벤트 실시간 집계 및 이상 탐지
    - 시계열 데이터 분석, 예측 모델 연계

---

## 2. 시스템 아키텍처 및 고가용성

```
[Log Source] → [Filebeat/Logstash(복수 인스턴스, 로드밸런싱)] → [Elasticsearch Cluster(3+ 노드, 샤드/레플리카)] → [Kibana(HA Proxy)]
```

- **분산 환경**: Elasticsearch 클러스터(3개 이상 노드), Logstash 멀티 인스턴스, Kibana HA 구성
- **고가용성(HA)**: 마스터 노드 최소 3개, 데이터 노드 다중화, Kibana 로드밸런싱
- **데이터 흐름**: 로그 수집 → 파싱/정제 → 저장/색인 → 시각화/분석

---

## 3. 설치 및 환경 구성 (고급)

### 3.1. Java 설치 (필요 시)
- ELK 스택은 Java를 필요로 할 수 있음 (버전에 따라 다름)
- OpenJDK 권장, JAVA_HOME 환경변수 설정

### 3.2. Elasticsearch 설치 및 클러스터 구성
```bash
# Linux 예시
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.x.x-linux-x86_64.tar.gz
tar -xzf elasticsearch-8.x.x-linux-x86_64.tar.gz
cd elasticsearch-8.x.x
# 클러스터 환경 설정 예시
cat <<EOF >> config/elasticsearch.yml
cluster.name: my-elk-cluster
node.name: node-1
network.host: 0.0.0.0
discovery.seed_hosts: ["node-1", "node-2", "node-3"]
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
EOF
./bin/elasticsearch
```
- **JVM/Heap 최적화**: `-Xms`, `-Xmx` 설정(메모리의 50% 권장, 최대 32GB)
- **보안 설정**: xpack.security.enabled, TLS/SSL 적용

### 3.3. Logstash 설치 및 멀티 파이프라인 구성
```bash
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.x.x-linux-x86_64.tar.gz
tar -xzf logstash-8.x.x-linux-x86_64.tar.gz
cd logstash-8.x.x
# 멀티 파이프라인 예시(logstash.yml)
pipeline:
  - pipeline.id: syslog
    path.config: "/etc/logstash/conf.d/syslog.conf"
  - pipeline.id: app
    path.config: "/etc/logstash/conf.d/app.conf"
./bin/logstash -f /etc/logstash/conf.d/
```
- **Persistent Queue**: 장애 시 데이터 유실 방지
- **Worker/Batch 튜닝**: throughput 최적화

### 3.4. Kibana 설치 및 고급 설정
```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.x.x-linux-x86_64.tar.gz
tar -xzf kibana-8.x.x-linux-x86_64.tar.gz
cd kibana-8.x.x
# kibana.yml 예시
elasticsearch.hosts: ["http://es-node-1:9200", "http://es-node-2:9200"]
server.host: "0.0.0.0"
server.publicBaseUrl: "https://kibana.example.com"
./bin/kibana
```
- **Role-Based Access Control**: 사용자별 대시보드 접근 권한 설정
- **Alerting/Watcher**: 실시간 알림 설정

---

## 4. Logstash 파이프라인 구성 예시 (고급)

### 4.1. Logstash 설정 파일(logstash.conf) 예시
```conf
input {
  beats {
    port => 5044
  }
  file {
    path => "/var/log/syslog"
    start_position => "beginning"
  }
}
filter {
  grok {
    match => { "message" => "%{SYSLOGTIMESTAMP:timestamp} %{SYSLOGHOST:host} %{DATA:program}: %{GREEDYDATA:log}" }
  }
  date {
    match => ["timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss"]
    target => "@timestamp"
  }
  mutate {
    remove_field => ["host", "path"]
  }
}
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "syslog-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "<비밀번호>"
    ssl => true
    cacert => "/etc/elasticsearch/certs/ca.crt"
  }
  stdout { codec => rubydebug }
}
```
- **고급 필터**: mutate, date, geoip, translate, aggregate 등 활용
- **보안 출력**: Elasticsearch 인증, TLS 적용

### 4.2. Logstash 실행
```bash
./bin/logstash -f logstash.conf
```
- **멀티 파이프라인**: 다양한 로그 소스별 파이프라인 분리
- **Persistent Queue**: 장애 시 데이터 유실 방지

---

## 5. Kibana를 통한 시각화 (고급)

1. **Kibana 접속**: 기본 포트는 http://localhost:5601, HTTPS 권장
2. **Index Pattern 생성**: 수집된 인덱스(`syslog-*` 등) 패턴을 등록
3. **Discover**: 로그 데이터 탐색, 필터/쿼리 활용
4. **Visualize**: Lens, Vega, TSVB 등 고급 시각화 생성
5. **Dashboard**: 여러 시각화 요소를 대시보드로 구성, 실시간 모니터링(Refresh Interval)
6. **Alerting**: Watcher, Kibana Alert로 실시간 알림 설정(Webhook, Slack 연동 등)
7. **Role-Based Access Control**: 사용자별 대시보드 접근 권한 설정

---

## 6. 보안 및 인증 (실전)

- **TLS/SSL 적용**: Elasticsearch, Logstash, Kibana 간 통신 암호화
- **사용자/역할 기반 접근 제어**: elasticsearch.yml, kibana.yml에서 security 설정
- **API Key 발급 및 활용**: REST API 호출 시 인증
- **Audit Logging**: 보안 이벤트 기록
- **Best Practice**: 기본 계정 비밀번호 변경, 최소 권한 원칙 적용

---

## 7. 성능 최적화 및 운영 자동화

- **Elasticsearch 인덱스 설계**: 템플릿, 롤오버, ILM(수명주기 관리) 활용
- **Logstash 파이프라인 튜닝**: 멀티 파이프라인, 큐, 워커, batch size 조정
- **JVM/Heap, ThreadPool 최적화**: 메모리, CPU 자원 효율적 사용
- **Beats 자동화**: Filebeat, Metricbeat 등으로 다양한 로그/메트릭 자동 수집
- **Elastic Stack Monitoring**: Stack 자체 상태 모니터링 및 알림
- **Snapshot & Restore**: 주기적 백업, 장애 시 복구

---

## 8. 실전 활용 예시 및 Best Practice

- **다양한 로그 포맷 파싱**: 웹서버, 애플리케이션, 시스템 로그 등
- **Kibana 고급 시각화**: 타임라인, Vega, Lens, TSVB 활용
- **실시간 알림**: Watcher, Webhook, Slack 연동
- **운영 체크리스트**: 인덱스 상태, 디스크 사용량, 노드 상태, 알림 설정 등
- **Best Practice**: 인덱스 롤오버, 템플릿 관리, 리소스 모니터링, 보안 강화

---

## 9. Troubleshooting & 운영 노하우

- **Elasticsearch Heap Full 해결**: JVM Heap 증설, 쿼리 최적화, 불필요한 인덱스 삭제
- **Logstash 파이프라인 지연 원인 분석**: 큐 적체, 필터 병목, 출력 지연 등 점검
- **Kibana 대시보드 최적화**: 시각화 최소화, 쿼리 효율화, 데이터 샘플링
- **운영 시 자주 발생하는 문제와 해결법**: 노드 장애, 인덱스 read-only, 인증 오류 등
- **실전 운영 경험**: 장애 복구, 확장/축소, 업그레이드, 보안 사고 대응 등

---

## 10. 참고 자료 및 커뮤니티
- [Elastic 공식 문서](https://www.elastic.co/guide/index.html)
- [ELK Stack Getting Started](https://www.elastic.co/what-is/elk-stack)
- [Elastic Discuss 커뮤니티](https://discuss.elastic.co/)
- [Awesome ELK Stack (GitHub)](https://github.com/donnemartin/awesome-elk)

---

## 11. 추가 팁 및 실전 노하우
- Filebeat, Metricbeat 등 Beats 계열을 활용하면 다양한 로그/메트릭 수집 가능
- 보안 및 인증(Elastic Stack Security) 설정 권장
- 대용량 환경에서는 클러스터 구성 및 리소스 튜닝 필요
- 운영 자동화: Ansible, Terraform 등 IaC 도구와 연계
- 실시간 모니터링 및 Alerting 체계 구축
- 장애 복구 및 백업 전략 수립 