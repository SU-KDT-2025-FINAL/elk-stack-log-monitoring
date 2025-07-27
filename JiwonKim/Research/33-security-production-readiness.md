# 3.3 보안 및 프로덕션 준비

## Overview
프로덕션 환경에서 ELK 스택을 안전하고 효율적으로 운영하기 위해서는 포괄적인 보안 설정과 운영 체계가 필요합니다. 이 문서에서는 보안 구현, 네트워크 보안, 운영 우수성을 다룹니다.

## 보안 구현

### X-Pack 보안 기능

#### 기본 보안 설정
```yaml
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.enrollment.enabled: true

# TLS 설정
xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12

# 암호 정책
xpack.security.authc.password_hashing.algorithm: bcrypt
```

#### 클러스터 간 통신 보안
```yaml
# 노드 간 통신 인증서 설정
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  client_authentication: required
  keystore.path: /etc/elasticsearch/certs/elastic-certificates.p12
  truststore.path: /etc/elasticsearch/certs/elastic-certificates.p12
```

### 사용자 인증 및 권한 부여

#### 내장 사용자 관리
```bash
# 슈퍼유저 비밀번호 설정
bin/elasticsearch-setup-passwords auto

# 커스텀 사용자 생성
PUT /_security/user/log_reader
{
  "password": "strong_password",
  "roles": ["kibana_user", "monitoring_user"],
  "full_name": "Log Reader User",
  "email": "logreader@company.com"
}
```

#### 외부 인증 시스템 연동
```yaml
# LDAP 연동
xpack.security.authc.realms.ldap.ldap1:
  order: 0
  url: "ldaps://ldap.company.com:636"
  bind_dn: "cn=ldapuser, ou=users, o=services, dc=company, dc=com"
  user_search:
    base_dn: "dc=company,dc=com"
    filter: "(cn={0})"
  group_search:
    base_dn: "dc=company,dc=com"
    filter: "(objectClass=posixGroup)"
  files:
    role_mapping: "/etc/elasticsearch/role_mapping.yml"
  ssl:
    certificate_authorities: ["/etc/ssl/certs/ca.crt"]
```

#### SAML 인증 설정
```yaml
xpack.security.authc.realms.saml.saml1:
  order: 2
  idp.metadata.path: saml/idp-metadata.xml
  idp.entity_id: "https://saml.company.com"
  sp.entity_id: "https://kibana.company.com"
  sp.acs: "https://kibana.company.com/api/security/saml/callback"
  sp.logout: "https://kibana.company.com/logout"
  attributes.principal: "http://saml.company.com/attributes/username"
  attributes.groups: "http://saml.company.com/attributes/groups"
```

### 역할 기반 접근 제어(RBAC)

#### 세밀한 권한 정의
```json
{
  "role_name": "application_analyst",
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["logs-application-*", "metrics-application-*"],
      "privileges": ["read", "view_index_metadata"],
      "field_security": {
        "grant": ["@timestamp", "message", "level", "service.*"],
        "except": ["user.password", "auth.token"]
      },
      "query": {
        "bool": {
          "filter": [
            {"term": {"environment": "production"}},
            {"range": {"@timestamp": {"gte": "now-30d"}}}
          ]
        }
      }
    }
  ],
  "applications": [
    {
      "application": "kibana-.kibana",
      "privileges": ["feature_discover.read", "feature_dashboard.read"],
      "resources": ["space:default"]
    }
  ]
}
```

#### 동적 역할 매핑
```yaml
# role_mapping.yml
admin:
  - "cn=elasticsearch-admins,dc=company,dc=com"
  - "admin"

developer:
  - "cn=developers,dc=company,dc=com"

analyst:
  - "cn=analysts,dc=company,dc=com"
```

### SSL/TLS 암호화 설정

#### 인증서 생성 및 관리
```bash
# CA 인증서 생성
bin/elasticsearch-certutil ca

# 노드 인증서 생성
bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12

# HTTP 인증서 생성
bin/elasticsearch-certutil http
```

#### Kibana SSL 설정
```yaml
# kibana.yml
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/kibana.crt
server.ssl.key: /etc/kibana/certs/kibana.key

elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/ca.crt"]
elasticsearch.ssl.verificationMode: certificate
```

#### Logstash SSL 설정
```ruby
# logstash.conf
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
    ssl_verify_mode => "force_peer"
  }
}

output {
  elasticsearch {
    hosts => ["https://es-node-1:9200"]
    ssl => true
    ssl_certificate_verification => true
    cacert => "/etc/logstash/certs/ca.crt"
    user => "logstash_writer"
    password => "${LOGSTASH_PASSWORD}"
  }
}
```

## 네트워크 보안

### 방화벽 구성

#### 포트 및 프로토콜 제한
```bash
# Elasticsearch 클러스터 통신 (9200, 9300)
iptables -A INPUT -p tcp --dport 9200 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 9300 -s 10.0.0.0/24 -j ACCEPT

# Kibana 웹 인터페이스 (5601)
iptables -A INPUT -p tcp --dport 5601 -s 10.0.1.0/24 -j ACCEPT

# Logstash Beats 입력 (5044)
iptables -A INPUT -p tcp --dport 5044 -s 10.0.2.0/24 -j ACCEPT

# 기본 거부
iptables -A INPUT -j DROP
```

#### 네트워크 세그멘테이션
```yaml
# Docker Compose 네트워크 예시
version: '3.8'
networks:
  elastic:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
  
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.21.0.0/16

services:
  elasticsearch:
    networks:
      - elastic
    
  kibana:
    networks:
      - elastic
      - frontend
```

### VPN 접근 제한

#### OpenVPN 연동 설정
```bash
# VPN 클라이언트 인증서 기반 접근
# /etc/openvpn/ccd/kibana-users
ifconfig-push 10.8.0.100 10.8.0.101
push "route 10.0.0.0 255.255.255.0"
```

#### IP 화이트리스트 관리
```json
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "zone",
    "network.host": ["_local_", "_site_"],
    "http.cors.enabled": false,
    "http.cors.allow-origin": "/https?:\\/\\/(localhost|kibana\\.company\\.com)(:[0-9]+)?/"
  }
}
```

### API 보안 모범 사례

#### API 키 관리
```bash
# API 키 생성
POST /_security/api_key
{
  "name": "monitoring_api_key",
  "role_descriptors": {
    "monitoring_role": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["logs-*"],
          "privileges": ["read"]
        }
      ]
    }
  },
  "expiration": "30d"
}
```

#### 요청 제한 및 레이트 리미팅
```json
{
  "persistent": {
    "indices.requests.cache.size": "20%",
    "indices.queries.cache.size": "10%",
    "search.max_buckets": 65536,
    "search.max_keep_alive": "5m"
  }
}
```

### 데이터 프라이버시 및 컴플라이언스

#### GDPR 컴플라이언스
```json
{
  "mappings": {
    "properties": {
      "user_data": {
        "type": "object",
        "enabled": false
      },
      "personal_info": {
        "type": "text",
        "store": false,
        "index": false
      },
      "anonymized_user_id": {
        "type": "keyword"
      }
    }
  }
}
```

#### 데이터 마스킹 및 익명화
```ruby
# Logstash 필터
filter {
  # 신용카드 번호 마스킹
  mutate {
    gsub => ["message", "\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b", "****-****-****-****"]
  }
  
  # 이메일 주소 해싱
  if [email] {
    fingerprint {
      source => "email"
      target => "email_hash"
      method => "SHA256"
      key => "secret_key"
    }
    mutate {
      remove_field => ["email"]
    }
  }
}
```

## 운영 우수성

### 클러스터 상태 모니터링

#### 헬스 체크 자동화
```bash
#!/bin/bash
# cluster_health_check.sh

CLUSTER_URL="https://elasticsearch:9200"
AUTH="elastic:password"

# 클러스터 상태 확인
HEALTH=$(curl -s -u $AUTH "$CLUSTER_URL/_cluster/health" | jq -r '.status')

if [ "$HEALTH" != "green" ]; then
  echo "ALERT: Cluster health is $HEALTH"
  # 알림 발송
  curl -X POST -H 'Content-type: application/json' \
    --data '{"text":"Elasticsearch cluster health: '$HEALTH'"}' \
    $SLACK_WEBHOOK_URL
fi

# 디스크 사용량 확인
DISK_USAGE=$(curl -s -u $AUTH "$CLUSTER_URL/_cat/allocation?v&h=disk.percent" | tail -n +2 | sort -n | tail -1)

if [ ${DISK_USAGE%?} -gt 85 ]; then
  echo "ALERT: High disk usage: $DISK_USAGE"
fi
```

#### 성능 메트릭 수집
```json
{
  "cluster_metrics": {
    "indices": {
      "indexing_rate": "indices.indexing.index_total",
      "search_rate": "indices.search.query_total",
      "query_latency": "indices.search.query_time_in_millis"
    },
    "nodes": {
      "cpu_usage": "os.cpu.percent",
      "memory_usage": "jvm.mem.heap_used_percent",
      "disk_usage": "fs.total.available_in_bytes"
    },
    "cluster": {
      "active_shards": "active_shards",
      "relocating_shards": "relocating_shards",
      "unassigned_shards": "unassigned_shards"
    }
  }
}
```

### 인덱스 최적화 및 유지보수

#### 자동 인덱스 관리
```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "7d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0,
            "include": {
              "box_type": "warm"
            }
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0,
            "include": {
              "box_type": "cold"
            }
          }
        }
      },
      "delete": {
        "min_age": "90d"
      }
    }
  }
}
```

#### 세그먼트 최적화
```bash
# 인덱스 세그먼트 정보 확인
GET /_cat/segments?v&h=index,shard,prirep,segment,size,size.memory

# 강제 병합 수행
POST /logs-2024.01.*/_forcemerge?max_num_segments=1
```

### 백업 및 재해 복구

#### 스냅샷 정책 설정
```json
{
  "policy": "daily_snapshots",
  "config": {
    "indices": ["logs-*", "metrics-*"],
    "ignore_unavailable": false,
    "include_global_state": false
  },
  "repository": "backup_repository",
  "schedule": {
    "cron": "0 2 * * *"
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

#### 복구 절차
```bash
# 스냅샷 복원
POST /_snapshot/backup_repository/snapshot_2024.01.15/_restore
{
  "indices": "logs-application-*",
  "ignore_unavailable": true,
  "include_global_state": false,
  "rename_pattern": "(.+)",
  "rename_replacement": "restored_$1"
}
```

### 리소스 사용률 분석

#### JVM 힙 모니터링
```bash
# 힙 사용률 모니터링 스크립트
#!/bin/bash
while true; do
  HEAP_USED=$(curl -s http://localhost:9200/_nodes/stats/jvm | jq '.nodes[].jvm.mem.heap_used_percent')
  echo "$(date): Heap usage: $HEAP_USED%"
  
  if [ $HEAP_USED -gt 85 ]; then
    echo "WARNING: High heap usage detected"
  fi
  
  sleep 60
done
```

#### 스레드 풀 모니터링
```json
{
  "nodes": {
    "thread_pool": {
      "search": {
        "queue_size": 1000,
        "size": 13,
        "active": 0,
        "rejected": 0
      },
      "write": {
        "queue_size": 200,
        "size": 8,
        "active": 2,
        "rejected": 0
      }
    }
  }
}
```

### 성능 벤치마킹

#### Rally를 이용한 성능 테스트
```bash
# Elasticsearch Rally 설치 및 실행
pip install esrally

# 벤치마크 실행
esrally race --track=geonames --target-hosts=localhost:9200 --report-file=benchmark_results.md
```

#### 커스텀 벤치마크
```python
# performance_test.py
import time
import requests
import json
from concurrent.futures import ThreadPoolExecutor

def index_document(doc_id):
    doc = {
        "@timestamp": "2024-01-15T10:00:00Z",
        "message": f"Test document {doc_id}",
        "level": "info"
    }
    
    response = requests.post(
        f"http://localhost:9200/test-index/_doc/{doc_id}",
        json=doc,
        auth=("elastic", "password")
    )
    return response.status_code

# 동시성 테스트
start_time = time.time()
with ThreadPoolExecutor(max_workers=10) as executor:
    results = list(executor.map(index_document, range(10000)))

end_time = time.time()
print(f"Indexed 10,000 documents in {end_time - start_time:.2f} seconds")
```

### 비용 최적화

#### 티어링 전략
```json
{
  "template": {
    "settings": {
      "index.routing.allocation.include._tier_preference": "data_hot",
      "index.lifecycle.name": "cost_optimized_policy"
    }
  }
}
```

#### 압축 및 스토리지 최적화
```json
{
  "settings": {
    "index.codec": "best_compression",
    "index.mapping.total_fields.limit": 1000,
    "index.refresh_interval": "30s",
    "index.number_of_replicas": 0
  }
}
```

## 문제 해결

### 일반적인 문제 및 해결법

#### 메모리 부족 문제
```bash
# JVM 힙 덤프 분석
jhat -J-Xmx4g heap_dump.hprof

# 메모리 사용량 최적화
echo 'indices.fielddata.cache.size: 20%' >> elasticsearch.yml
echo 'indices.requests.cache.size: 1%' >> elasticsearch.yml
```

#### 인덱싱 성능 저하
```json
{
  "settings": {
    "index.refresh_interval": "30s",
    "index.number_of_replicas": 0,
    "index.translog.flush_threshold_size": "1gb",
    "index.merge.scheduler.max_thread_count": 1
  }
}
```

#### 검색 성능 문제
```json
{
  "query": {
    "bool": {
      "filter": [
        {"range": {"@timestamp": {"gte": "now-1h"}}},
        {"term": {"level": "error"}}
      ]
    }
  }
}
```

## Best Practices

### 보안 관리
- **최소 권한 원칙**: 필요한 최소한의 권한만 부여
- **정기 감사**: 사용자 권한 및 접근 로그 정기 검토
- **암호 정책**: 강력한 암호 정책 및 정기 변경
- **보안 업데이트**: 정기적인 보안 패치 및 업데이트

### 운영 관리
- **모니터링**: 포괄적인 시스템 모니터링 체계 구축
- **문서화**: 운영 절차 및 장애 대응 매뉴얼 작성
- **교육**: 운영팀 대상 정기 교육 및 훈련
- **테스트**: 정기적인 재해 복구 테스트

### 성능 관리
- **용량 계획**: 데이터 증가율 기반 용량 계획 수립
- **리소스 최적화**: CPU, 메모리, 디스크 사용률 최적화
- **네트워크 최적화**: 클러스터 간 통신 최적화
- **인덱스 설계**: 효율적인 인덱스 구조 설계

## Benefits and Challenges

### Benefits
- **보안성**: 강력한 인증, 권한 부여, 암호화 기능
- **확장성**: 대규모 프로덕션 환경 지원
- **신뢰성**: 고가용성 및 재해 복구 기능
- **관리성**: 자동화된 운영 및 모니터링 도구

### Challenges
- **복잡성**: 보안 설정 및 운영 절차의 복잡성
- **비용**: 라이선스 및 인프라 비용
- **전문성**: 전문적인 운영 지식 요구
- **규정 준수**: 다양한 규정 및 컴플라이언스 요구사항