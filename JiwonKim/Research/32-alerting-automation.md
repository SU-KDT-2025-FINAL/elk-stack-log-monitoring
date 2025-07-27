# 3.2 알림 및 자동화

## Overview
Elasticsearch Watcher와 Kibana Alerting은 로그 데이터의 이상 징후를 실시간으로 감지하고 자동화된 대응을 제공합니다. 이 문서에서는 효과적인 알림 체계 구축과 자동화 전략을 다룹니다.

## Watcher 구성

### 기본 Watcher 구조
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
        "search_type": "query_then_fetch",
        "indices": ["logs-*"],
        "body": {
          "query": {
            "bool": {
              "filter": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "error_count": {
              "filter": {
                "term": {
                  "level": "error"
                }
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.aggregations.error_count.doc_count": {
        "gt": 10
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": ["ops@company.com"],
        "subject": "High Error Rate Detected",
        "body": "Error count: {{ctx.payload.aggregations.error_count.doc_count}}"
      }
    }
  }
}
```

### 고급 트리거 설정

#### 시간 기반 트리거
```json
{
  "trigger": {
    "schedule": {
      "cron": "0 */5 * * * ?"
    }
  }
}

{
  "trigger": {
    "schedule": {
      "daily": {
        "at": ["09:00", "17:00"]
      }
    }
  }
}

{
  "trigger": {
    "schedule": {
      "weekly": [
        {"on": ["monday", "wednesday", "friday"], "at": ["09:00"]}
      ]
    }
  }
}
```

#### 동적 트리거
```json
{
  "trigger": {
    "schedule": {
      "interval": "{{ctx.metadata.check_interval}}"
    }
  },
  "metadata": {
    "check_interval": "30s",
    "escalation_level": 1
  }
}
```

### 알림 규칙 생성 및 관리

#### 임계값 기반 알림
```json
{
  "watch_id": "cpu_usage_alert",
  "input": {
    "search": {
      "request": {
        "indices": ["metricbeat-*"],
        "body": {
          "query": {
            "bool": {
              "filter": [
                {"range": {"@timestamp": {"gte": "now-5m"}}},
                {"term": {"metricset.name": "cpu"}}
              ]
            }
          },
          "aggs": {
            "avg_cpu_usage": {
              "avg": {
                "field": "system.cpu.total.pct"
              }
            },
            "hosts": {
              "terms": {
                "field": "host.name",
                "size": 100
              },
              "aggs": {
                "avg_cpu": {
                  "avg": {
                    "field": "system.cpu.total.pct"
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": """
        def threshold = 0.8;
        def high_cpu_hosts = [];
        
        for (host in ctx.payload.aggregations.hosts.buckets) {
          if (host.avg_cpu.value > threshold) {
            high_cpu_hosts.add(host.key);
          }
        }
        
        ctx.vars.high_cpu_hosts = high_cpu_hosts;
        return high_cpu_hosts.size() > 0;
      """
    }
  }
}
```

#### 패턴 기반 알림
```json
{
  "watch_id": "error_pattern_detection",
  "input": {
    "search": {
      "request": {
        "indices": ["logs-application-*"],
        "body": {
          "query": {
            "bool": {
              "filter": [
                {"range": {"@timestamp": {"gte": "now-10m"}}},
                {"term": {"level": "error"}}
              ]
            }
          },
          "aggs": {
            "error_patterns": {
              "terms": {
                "script": {
                  "source": "doc['message.keyword'].value.substring(0, Math.min(50, doc['message.keyword'].value.length()))"
                },
                "size": 10
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": """
        for (pattern in ctx.payload.aggregations.error_patterns.buckets) {
          if (pattern.doc_count > 5) {
            return true;
          }
        }
        return false;
      """
    }
  }
}
```

### 조건부 알림

#### 복합 조건 설정
```json
{
  "condition": {
    "script": {
      "source": """
        def error_rate = ctx.payload.aggregations.error_count.doc_count / ctx.payload.aggregations.total_count.doc_count;
        def response_time = ctx.payload.aggregations.avg_response_time.value;
        
        // 에러율이 5% 이상이거나 응답시간이 2초 이상인 경우
        return error_rate > 0.05 || response_time > 2000;
      """
    }
  }
}
```

#### 시간대별 동적 임계값
```json
{
  "condition": {
    "script": {
      "source": """
        def hour = ZonedDateTime.now().getHour();
        def threshold;
        
        // 업무시간(9-18시)과 비업무시간의 다른 임계값 적용
        if (hour >= 9 && hour <= 18) {
          threshold = 100; // 업무시간: 높은 임계값
        } else {
          threshold = 50;  // 비업무시간: 낮은 임계값
        }
        
        return ctx.payload.aggregations.error_count.doc_count > threshold;
      """
    }
  }
}
```

### 다양한 액션 타입

#### 이메일 알림
```json
{
  "actions": {
    "send_email": {
      "email": {
        "profile": "standard",
        "to": ["ops@company.com", "dev@company.com"],
        "cc": ["manager@company.com"],
        "subject": "{{ctx.watch_id}} Alert - {{ctx.execution_time}}",
        "body": {
          "html": """
            <h2>Alert: {{ctx.watch_id}}</h2>
            <p><strong>Time:</strong> {{ctx.execution_time}}</p>
            <p><strong>Error Count:</strong> {{ctx.payload.aggregations.error_count.doc_count}}</p>
            <p><strong>Affected Hosts:</strong></p>
            <ul>
            {{#ctx.vars.high_cpu_hosts}}
              <li>{{.}}</li>
            {{/ctx.vars.high_cpu_hosts}}
            </ul>
          """
        },
        "attachments": {
          "data.json": {
            "data": {
              "format": "json"
            }
          }
        }
      }
    }
  }
}
```

#### Slack 통합
```json
{
  "actions": {
    "send_slack": {
      "webhook": {
        "scheme": "https",
        "host": "hooks.slack.com",
        "port": 443,
        "method": "post",
        "path": "/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX",
        "headers": {
          "Content-Type": "application/json"
        },
        "body": """
        {
          "channel": "#alerts",
          "username": "ElasticSearch Watcher",
          "text": "🚨 *{{ctx.watch_id}}* Alert",
          "attachments": [
            {
              "color": "danger",
              "fields": [
                {
                  "title": "Error Count",
                  "value": "{{ctx.payload.aggregations.error_count.doc_count}}",
                  "short": true
                },
                {
                  "title": "Time",
                  "value": "{{ctx.execution_time}}",
                  "short": true
                }
              ]
            }
          ]
        }
        """
      }
    }
  }
}
```

#### 웹훅 연동
```json
{
  "actions": {
    "webhook_action": {
      "webhook": {
        "scheme": "https",
        "host": "api.company.com",
        "port": 443,
        "method": "post",
        "path": "/alerts",
        "headers": {
          "Content-Type": "application/json",
          "Authorization": "Bearer {{ctx.metadata.api_token}}"
        },
        "body": """
        {
          "alert_id": "{{ctx.watch_id}}",
          "timestamp": "{{ctx.execution_time}}",
          "severity": "high",
          "metrics": {
            "error_count": {{ctx.payload.aggregations.error_count.doc_count}},
            "error_rate": {{ctx.vars.error_rate}}
          },
          "affected_services": {{#toJson}}ctx.vars.affected_services{{/toJson}}
        }
        """
      }
    }
  }
}
```

### 에스컬레이션 정책

#### 단계별 에스컬레이션
```json
{
  "watch_id": "escalation_example",
  "metadata": {
    "escalation_level": 1,
    "last_alert_time": null,
    "alert_count": 0
  },
  "condition": {
    "script": {
      "source": """
        def alert_threshold = 10;
        def current_errors = ctx.payload.aggregations.error_count.doc_count;
        
        if (current_errors > alert_threshold) {
          ctx.vars.current_escalation = ctx.metadata.escalation_level;
          ctx.vars.alert_needed = true;
          
          // 에스컬레이션 레벨 증가
          if (ctx.metadata.alert_count > 3) {
            ctx.vars.current_escalation = Math.min(3, ctx.metadata.escalation_level + 1);
          }
          
          return true;
        }
        return false;
      """
    }
  },
  "actions": {
    "level_1_alert": {
      "condition": {
        "compare": {
          "ctx.vars.current_escalation": {
            "eq": 1
          }
        }
      },
      "email": {
        "to": ["oncall@company.com"],
        "subject": "Level 1 Alert: High Error Rate"
      }
    },
    "level_2_alert": {
      "condition": {
        "compare": {
          "ctx.vars.current_escalation": {
            "eq": 2
          }
        }
      },
      "email": {
        "to": ["oncall@company.com", "team-lead@company.com"],
        "subject": "Level 2 ESCALATED Alert: Persistent High Error Rate"
      }
    },
    "level_3_alert": {
      "condition": {
        "compare": {
          "ctx.vars.current_escalation": {
            "eq": 3
          }
        }
      },
      "email": {
        "to": ["oncall@company.com", "team-lead@company.com", "manager@company.com"],
        "subject": "Level 3 CRITICAL ESCALATED Alert: System Emergency"
      }
    }
  }
}
```

### 억제 규칙

#### 시간 기반 억제
```json
{
  "throttle_period": "15m",
  "throttle_period_in_millis": 900000
}
```

#### 조건부 억제
```json
{
  "actions": {
    "send_alert": {
      "condition": {
        "script": {
          "source": """
            // 마지막 알림으로부터 30분이 지났거나, 에러 수가 2배 이상 증가한 경우만 알림
            def last_alert = ctx.metadata.last_alert_time;
            def current_time = ctx.execution_time.getMillis();
            def current_errors = ctx.payload.aggregations.error_count.doc_count;
            def last_errors = ctx.metadata.last_error_count;
            
            if (last_alert == null) {
              return true; // 첫 번째 알림
            }
            
            def time_diff = current_time - last_alert;
            if (time_diff > 1800000) { // 30분
              return true;
            }
            
            if (current_errors > last_errors * 2) {
              return true; // 에러 수 급증
            }
            
            return false;
          """
        }
      }
    }
  }
}
```

## 자동화된 응답 시스템

### 트리거 기반 액션

#### 자동 복구 스크립트
```json
{
  "actions": {
    "auto_restart_service": {
      "webhook": {
        "scheme": "https",
        "host": "automation.company.com",
        "path": "/api/restart-service",
        "method": "post",
        "headers": {
          "Content-Type": "application/json",
          "Authorization": "Bearer {{ctx.metadata.automation_token}}"
        },
        "body": """
        {
          "action": "restart",
          "service": "{{ctx.vars.failed_service}}",
          "host": "{{ctx.vars.affected_host}}",
          "reason": "High error rate detected",
          "alert_id": "{{ctx.watch_id}}-{{ctx.execution_time}}"
        }
        """
      }
    }
  }
}
```

#### 동적 스케일링
```json
{
  "actions": {
    "scale_up": {
      "condition": {
        "script": {
          "source": """
            def cpu_usage = ctx.payload.aggregations.avg_cpu.value;
            def memory_usage = ctx.payload.aggregations.avg_memory.value;
            
            return cpu_usage > 0.8 || memory_usage > 0.85;
          """
        }
      },
      "webhook": {
        "host": "k8s-api.company.com",
        "path": "/api/v1/namespaces/production/deployments/{{ctx.vars.service_name}}/scale",
        "method": "patch",
        "body": """
        {
          "spec": {
            "replicas": {{ctx.vars.current_replicas + 2}}
          }
        }
        """
      }
    }
  }
}
```

### 성능 임계값 모니터링

#### 다중 메트릭 모니터링
```json
{
  "input": {
    "search": {
      "request": {
        "indices": ["metricbeat-*", "logs-*"],
        "body": {
          "query": {
            "bool": {
              "filter": [
                {"range": {"@timestamp": {"gte": "now-5m"}}}
              ]
            }
          },
          "aggs": {
            "performance_metrics": {
              "multi_terms": {
                "terms": [
                  {"field": "host.name"},
                  {"field": "service.name"}
                ]
              },
              "aggs": {
                "avg_cpu": {
                  "avg": {"field": "system.cpu.total.pct"}
                },
                "avg_memory": {
                  "avg": {"field": "system.memory.usage.pct"}
                },
                "avg_response_time": {
                  "avg": {"field": "response_time"}
                },
                "error_rate": {
                  "filter": {"term": {"level": "error"}},
                  "aggs": {
                    "error_percentage": {
                      "bucket_script": {
                        "buckets_path": {
                          "errors": "_count",
                          "total": "_parent>_count"
                        },
                        "script": "params.errors / params.total * 100"
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### 사용자 정의 알림 로직

#### 비즈니스 메트릭 기반 알림
```json
{
  "watch_id": "business_metrics_alert",
  "input": {
    "search": {
      "request": {
        "indices": ["business-events-*"],
        "body": {
          "query": {
            "bool": {
              "filter": [
                {"range": {"@timestamp": {"gte": "now-1h"}}},
                {"term": {"event_type": "transaction"}}
              ]
            }
          },
          "aggs": {
            "revenue": {
              "sum": {"field": "amount"}
            },
            "transaction_count": {
              "value_count": {"field": "transaction_id"}
            },
            "failed_transactions": {
              "filter": {"term": {"status": "failed"}},
              "aggs": {
                "count": {"value_count": {"field": "transaction_id"}}
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": """
        def revenue = ctx.payload.aggregations.revenue.value;
        def total_transactions = ctx.payload.aggregations.transaction_count.value;
        def failed_transactions = ctx.payload.aggregations.failed_transactions.count.value;
        
        def failure_rate = failed_transactions / total_transactions;
        def revenue_per_hour = revenue;
        
        // 실패율이 5% 이상이거나 시간당 매출이 목표의 80% 미만인 경우
        def target_revenue_per_hour = 50000;
        
        return failure_rate > 0.05 || revenue_per_hour < (target_revenue_per_hour * 0.8);
      """
    }
  }
}
```

## Kibana Alerting

### 규칙 기반 알림
```json
{
  "rule": {
    "name": "High CPU Usage",
    "rule_type_id": ".index-threshold",
    "params": {
      "index": ["metricbeat-*"],
      "timeField": "@timestamp",
      "aggType": "avg",
      "aggField": "system.cpu.total.pct",
      "termField": "host.name",
      "termSize": 10,
      "timeWindowSize": 5,
      "timeWindowUnit": "m",
      "thresholdComparator": ">",
      "threshold": [0.8]
    },
    "consumer": "alerts",
    "schedule": {
      "interval": "1m"
    },
    "actions": [
      {
        "id": "email-action",
        "group": "threshold met",
        "params": {
          "to": ["admin@company.com"],
          "subject": "High CPU Alert: {{context.group}}",
          "message": "CPU usage is {{context.value}} on host {{context.group}}"
        }
      }
    ]
  }
}
```

## Best Practices

### 알림 설계 원칙
- **신호 대 노이즈 비율**: 중요한 알림만 발송하여 알림 피로도 방지
- **계층적 알림**: 심각도에 따른 단계별 알림 체계 구축
- **자동화 우선**: 단순 반복 작업은 자동화로 해결
- **모니터링 모니터링**: 알림 시스템 자체의 상태 모니터링

### 성능 최적화
- **효율적인 쿼리**: 인덱스 패턴과 시간 범위 최적화
- **적절한 실행 주기**: 알림 중요도에 따른 실행 빈도 조정
- **리소스 관리**: Watcher 실행으로 인한 클러스터 부하 관리
- **데이터 보존**: 알림 히스토리 데이터 적절한 보존 정책

### 운영 관리
- **문서화**: 알림 규칙과 대응 절차 문서화
- **테스트**: 정기적인 알림 시스템 테스트
- **검토**: 알림 효과성 정기 검토 및 개선
- **교육**: 팀원 대상 알림 시스템 교육

## Benefits and Challenges

### Benefits
- **신속한 대응**: 문제 발생 시 즉각적인 알림과 대응
- **자동화**: 반복적인 운영 작업의 자동화
- **예방적 관리**: 문제 발생 전 사전 감지 및 대응
- **확장성**: 다양한 메트릭과 시나리오에 대한 유연한 대응

### Challenges
- **복잡성**: 복잡한 알림 로직 설계 및 관리
- **오탐/미탐**: 적절한 임계값 설정의 어려움
- **통합 관리**: 다양한 알림 채널의 통합 관리
- **성능 영향**: 알림 시스템이 주 시스템에 미치는 성능 영향