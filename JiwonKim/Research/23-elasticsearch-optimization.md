# 2.3 Elasticsearch 최적화

## Overview
Elasticsearch는 분산 검색 및 분석 엔진으로, 대용량 데이터 처리를 위해서는 적절한 최적화가 필수입니다. 이 문서에서는 인덱스 관리, 매핑 최적화, 성능 튜닝 기법을 다룹니다.

## Elasticsearch 고급 기능

### 인덱스 템플릿 및 매핑

#### 동적 템플릿 활용
```json
{
  "index_patterns": ["logs-*"],
  "template": {
    "mappings": {
      "dynamic_templates": [
        {
          "strings_as_keywords": {
            "match_mapping_type": "string",
            "match": "*_id",
            "mapping": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        {
          "strings_as_text": {
            "match_mapping_type": "string",
            "mapping": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",
                  "ignore_above": 256
                }
              }
            }
          }
        }
      ],
      "properties": {
        "@timestamp": {
          "type": "date",
          "format": "strict_date_optional_time||epoch_millis"
        },
        "message": {
          "type": "text",
          "analyzer": "standard"
        },
        "level": {
          "type": "keyword"
        },
        "response_time": {
          "type": "float"
        }
      }
    },
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "refresh_interval": "30s",
      "index.mapping.total_fields.limit": 2000
    }
  }
}
```

#### 컴포넌트 템플릿 구성
```json
{
  "component_templates": {
    "logs_mappings": {
      "template": {
        "mappings": {
          "properties": {
            "@timestamp": {"type": "date"},
            "log": {
              "properties": {
                "level": {"type": "keyword"},
                "logger": {"type": "keyword"}
              }
            }
          }
        }
      }
    },
    "logs_settings": {
      "template": {
        "settings": {
          "number_of_shards": 1,
          "number_of_replicas": 1,
          "refresh_interval": "30s"
        }
      }
    }
  },
  "index_templates": {
    "logs_template": {
      "index_patterns": ["logs-*"],
      "composed_of": ["logs_mappings", "logs_settings"],
      "priority": 200
    }
  }
}
```

### 분석기 및 토크나이저

#### 커스텀 분석기 정의
```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "custom_log_analyzer": {
          "type": "custom",
          "tokenizer": "keyword",
          "filter": [
            "lowercase",
            "custom_stopwords",
            "custom_synonyms"
          ]
        },
        "path_analyzer": {
          "type": "custom",
          "tokenizer": "path_hierarchy",
          "filter": ["lowercase"]
        }
      },
      "tokenizer": {
        "custom_pattern": {
          "type": "pattern",
          "pattern": "[\\W&&[^/]]+"
        }
      },
      "filter": {
        "custom_stopwords": {
          "type": "stop",
          "stopwords": ["the", "is", "at", "which", "on"]
        },
        "custom_synonyms": {
          "type": "synonym",
          "synonyms": [
            "error,err,failure",
            "warning,warn,alert"
          ]
        }
      }
    }
  }
}
```

#### 다국어 분석기
```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "multilang_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "cjk_width",
            "korean_stop",
            "korean_stemmer"
          ]
        }
      },
      "filter": {
        "korean_stop": {
          "type": "stop",
          "stopwords": "_korean_"
        },
        "korean_stemmer": {
          "type": "stemmer",
          "language": "light_korean"
        }
      }
    }
  }
}
```

### Query DSL 및 집계

#### 복합 쿼리 최적화
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h",
              "lte": "now"
            }
          }
        }
      ],
      "filter": [
        {"term": {"level": "error"}},
        {"exists": {"field": "user.id"}}
      ],
      "should": [
        {"match": {"message": "database"}},
        {"match": {"message": "connection"}}
      ],
      "minimum_should_match": 1
    }
  },
  "aggs": {
    "error_trends": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "5m"
      },
      "aggs": {
        "top_errors": {
          "terms": {
            "field": "error.type",
            "size": 10
          }
        }
      }
    }
  }
}
```

#### 고급 집계 활용
```json
{
  "aggs": {
    "response_time_stats": {
      "extended_stats": {
        "field": "response_time"
      }
    },
    "percentiles_response": {
      "percentiles": {
        "field": "response_time",
        "percents": [50, 75, 90, 95, 99]
      }
    },
    "significant_terms": {
      "significant_terms": {
        "field": "user_agent.keyword",
        "size": 10
      }
    },
    "nested_agg": {
      "nested": {
        "path": "errors"
      },
      "aggs": {
        "error_types": {
          "terms": {
            "field": "errors.type"
          }
        }
      }
    }
  }
}
```

### 검색 최적화 기법

#### 필드 레벨 부스팅
```json
{
  "query": {
    "multi_match": {
      "query": "database error",
      "fields": [
        "message^3",
        "error.message^2",
        "stack_trace"
      ],
      "type": "best_fields"
    }
  }
}
```

#### 스크립트 쿼리 최적화
```json
{
  "query": {
    "script": {
      "script": {
        "source": "doc['response_time'].value > params.threshold",
        "params": {
          "threshold": 1000
        }
      }
    }
  }
}
```

## 인덱스 관리

### 인덱스 라이프사이클 관리(ILM)

#### ILM 정책 정의
```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "7d",
            "max_docs": 100000000
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "set_priority": {
            "priority": 50
          },
          "allocate": {
            "number_of_replicas": 0
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "shrink": {
            "number_of_shards": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "set_priority": {
            "priority": 0
          },
          "allocate": {
            "number_of_replicas": 0
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

#### 롤오버 인덱스 설정
```json
{
  "aliases": {
    "logs-write": {
      "is_write_index": true
    },
    "logs": {}
  },
  "settings": {
    "index.lifecycle.name": "logs_policy",
    "index.lifecycle.rollover_alias": "logs-write"
  }
}
```

### 샤드 할당 및 복제본 구성

#### 샤드 할당 정책
```json
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "rack_id",
    "cluster.routing.allocation.balance.shard": 0.45,
    "cluster.routing.allocation.balance.index": 0.55,
    "cluster.routing.allocation.balance.threshold": 1.0
  }
}
```

#### 인덱스별 라우팅 설정
```json
{
  "settings": {
    "index.routing.allocation.include.box_type": "hot",
    "index.routing.allocation.require.data": "true",
    "index.routing.allocation.exclude._ip": "10.0.0.1"
  }
}
```

### 저장소 최적화 기법

#### 압축 및 스토리지 최적화
```json
{
  "settings": {
    "index.codec": "best_compression",
    "index.store.type": "fs",
    "index.merge.policy.max_merged_segment": "5gb",
    "index.merge.scheduler.max_thread_count": 1
  }
}
```

#### 필드 레벨 압축
```json
{
  "mappings": {
    "properties": {
      "large_text_field": {
        "type": "text",
        "store": true,
        "compress": true
      },
      "binary_data": {
        "type": "binary",
        "compress": true
      }
    }
  }
}
```

## 클러스터 최적화

### 노드 역할 최적화
```yaml
# elasticsearch.yml
node.roles: [master, data, ingest]
node.attr.box_type: hot
node.attr.rack_id: rack1

# 마스터 전용 노드
node.roles: [master]
node.master: true
node.data: false
node.ingest: false

# 데이터 전용 노드  
node.roles: [data]
node.master: false
node.data: true
node.ingest: false
```

### 메모리 및 JVM 튜닝
```yaml
# jvm.options
-Xms16g
-Xmx16g
-XX:+UseG1GC
-XX:G1HeapRegionSize=16m
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+UseStringDeduplication

# 힙 덤프 설정
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/lib/elasticsearch/heap_dumps/

# GC 로깅
-Xlog:gc*,gc+age=trace,safepoint:gc.log:time,level,tags
```

### 스레드 풀 튜닝
```json
{
  "persistent": {
    "thread_pool.search.size": 13,
    "thread_pool.search.queue_size": 1000,
    "thread_pool.write.size": 8,
    "thread_pool.write.queue_size": 200
  }
}
```

## 성능 모니터링

### 클러스터 상태 모니터링
```bash
# 클러스터 헬스 체크
GET /_cluster/health

# 노드 통계
GET /_nodes/stats

# 인덱스 통계
GET /_stats

# 세그먼트 정보
GET /_cat/segments?v&h=index,shard,prirep,segment,size,size.memory
```

### 성능 메트릭 수집
```json
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-5m"
      }
    }
  },
  "aggs": {
    "search_latency": {
      "avg": {
        "field": "search.query_time_in_millis"
      }
    },
    "indexing_rate": {
      "rate": {
        "field": "indices.indexing.index_total"
      }
    }
  }
}
```

## Best Practices

### 인덱스 설계
- **적절한 샤드 수**: 노드당 샤드 수를 20-25개로 제한
- **샤드 크기**: 샤드당 20-40GB 크기 유지
- **매핑 최적화**: 불필요한 필드 비활성화, 적절한 데이터 타입 선택
- **동적 매핑 제한**: 필드 수 제한으로 매핑 폭발 방지

### 쿼리 최적화
- **필터 컨텍스트 활용**: 점수 계산이 불필요한 경우 필터 사용
- **집계 최적화**: 날짜 히스토그램 간격 조정, 샘플링 활용
- **캐싱 활용**: 자주 사용되는 쿼리는 캐시 효과 극대화
- **스크롤 대신 Search After**: 대량 데이터 처리 시 성능 개선

### 운영 관리
- **모니터링**: 클러스터 상태, 인덱싱 속도, 검색 지연 시간 추적
- **백업**: 스냅샷 주기적 생성 및 복원 테스트
- **업그레이드**: 롤링 업그레이드를 통한 무중단 버전 업데이트
- **용량 계획**: 데이터 증가율 모니터링 및 확장 계획 수립

## Benefits and Challenges

### Benefits
- **확장성**: 수평적 확장을 통한 대용량 데이터 처리
- **성능**: 분산 아키텍처와 최적화를 통한 고성능 검색
- **유연성**: 다양한 데이터 타입과 쿼리 패턴 지원
- **실시간성**: 준실시간 인덱싱과 검색 기능

### Challenges
- **복잡성**: 클러스터 관리와 최적화의 복잡성
- **리소스 요구**: 높은 메모리와 CPU 요구사항
- **데이터 일관성**: 분산 환경에서의 일관성 관리
- **운영 비용**: 인프라 및 운영 비용 최적화 필요