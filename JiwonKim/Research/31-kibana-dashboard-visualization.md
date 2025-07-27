# 3.1 Kibana 대시보드 및 시각화

## Overview
Kibana는 Elasticsearch 데이터를 시각화하고 분석하는 강력한 도구입니다. 이 문서에서는 고급 시각화 기법, 대시보드 구성, 실시간 모니터링 설정 방법을 다룹니다.

## Kibana 고급 기능

### 커스텀 대시보드 생성 및 관리

#### 대시보드 구조 설계
```json
{
  "dashboard": {
    "title": "Application Performance Monitoring",
    "description": "실시간 애플리케이션 성능 모니터링 대시보드",
    "panels": [
      {
        "type": "line_chart",
        "title": "Response Time Trend",
        "timeframe": "last_24h",
        "refresh_interval": "30s"
      },
      {
        "type": "metric",
        "title": "Error Rate",
        "alert_threshold": 5,
        "format": "percentage"
      }
    ],
    "filters": {
      "global": [
        {"field": "environment", "value": "production"},
        {"field": "service", "values": ["api", "web", "worker"]}
      ]
    }
  }
}
```

#### 대시보드 템플릿 활용
```json
{
  "template": {
    "name": "service_monitoring_template",
    "variables": [
      {
        "name": "service_name",
        "type": "text",
        "default": "api"
      },
      {
        "name": "time_range",
        "type": "timefilter",
        "default": "now-1h"
      }
    ],
    "panels": [
      {
        "title": "{{service_name}} Error Rate",
        "query": "service:{{service_name}} AND level:error"
      }
    ]
  }
}
```

### KQL(Kibana Query Language) 마스터

#### 기본 KQL 문법
```kql
# 필드 검색
status: 200
response_time > 1000
user.id: "12345"

# 와일드카드 검색
message: *error*
path: /api/*

# 범위 검색
@timestamp >= "2023-01-01" and @timestamp < "2023-02-01"
bytes: [1000 TO 5000]

# 불린 연산자
status: 200 AND response_time > 1000
level: error OR level: warning
NOT status: 404

# 존재 여부 확인
user.id: *
NOT _exists_: error.message
```

#### 복합 KQL 쿼리
```kql
# 복잡한 조건 조합
(status: 500 OR status: 502) AND 
service: "api" AND 
@timestamp >= now-1h AND
user.id: * AND
NOT path: "/health"

# 정규식 활용
message: /error|exception|fail/
path: /\/api\/v[0-9]+\/.*/

# 중첩 필드 검색
user: { id: "12345" AND role: "admin" }
error: { type: "database" AND code: 1045 }
```

### 다양한 시각화 유형

#### 라인 차트를 통한 시계열 분석
```json
{
  "visualization": {
    "type": "line",
    "title": "API Response Time Trend",
    "data_source": {
      "index": "logs-api-*",
      "query": "service:api AND response_time:*"
    },
    "x_axis": {
      "field": "@timestamp",
      "interval": "5m"
    },
    "y_axis": [
      {
        "field": "response_time",
        "aggregation": "avg",
        "label": "Average Response Time"
      },
      {
        "field": "response_time",
        "aggregation": "percentiles",
        "percentiles": [95],
        "label": "95th Percentile"
      }
    ],
    "breakdown": {
      "field": "service.name",
      "top": 5
    }
  }
}
```

#### 바 차트를 통한 분포 분석
```json
{
  "visualization": {
    "type": "vertical_bar",
    "title": "HTTP Status Code Distribution",
    "data_source": {
      "index": "logs-nginx-*",
      "time_field": "@timestamp"
    },
    "x_axis": {
      "field": "status",
      "type": "terms",
      "size": 10
    },
    "y_axis": {
      "aggregation": "count",
      "label": "Request Count"
    },
    "color_mapping": {
      "2xx": "#00BFA5",
      "3xx": "#FFC107", 
      "4xx": "#FF9800",
      "5xx": "#F44336"
    }
  }
}
```

#### 히트맵을 통한 패턴 인식
```json
{
  "visualization": {
    "type": "heatmap",
    "title": "Hourly Request Pattern",
    "data_source": {
      "index": "logs-access-*"
    },
    "x_axis": {
      "field": "@timestamp",
      "interval": "1h"
    },
    "y_axis": {
      "field": "path.keyword",
      "type": "terms",
      "size": 20
    },
    "value": {
      "aggregation": "count"
    },
    "color_scale": {
      "type": "linear",
      "colors": ["#ffffff", "#ff0000"]
    }
  }
}
```

#### 지도를 통한 지리적 데이터
```json
{
  "visualization": {
    "type": "coordinate_map",
    "title": "Global Traffic Distribution",
    "data_source": {
      "index": "logs-web-*",
      "query": "geoip.location:*"
    },
    "map_settings": {
      "center": [0, 0],
      "zoom": 2,
      "base_layer": "road_map"
    },
    "layers": [
      {
        "type": "tile_map",
        "field": "geoip.location",
        "aggregation": "count",
        "color_schema": "Yellow to Red"
      }
    ]
  }
}
```

### 고급 시각화 도구

#### Lens 활용
```json
{
  "lens_visualization": {
    "type": "lnsXY",
    "title": "Multi-metric Analysis",
    "layers": [
      {
        "layerId": "layer1",
        "layerType": "data",
        "seriesType": "line",
        "accessors": ["response_time_avg"],
        "splitAccessor": "service.name",
        "xAccessor": "@timestamp"
      }
    ],
    "datasourceStates": {
      "indexpattern": {
        "layers": {
          "layer1": {
            "columns": {
              "@timestamp": {
                "dataType": "date",
                "isBucketed": true,
                "operationType": "date_histogram",
                "params": {"interval": "auto"}
              },
              "response_time_avg": {
                "dataType": "number",
                "isBucketed": false,
                "operationType": "average",
                "sourceField": "response_time"
              }
            }
          }
        }
      }
    }
  }
}
```

#### Vega 커스텀 시각화
```json
{
  "vega_spec": {
    "$schema": "https://vega.github.io/schema/vega-lite/v4.json",
    "data": {
      "url": {
        "index": "logs-*",
        "body": {
          "aggs": {
            "time_buckets": {
              "date_histogram": {
                "field": "@timestamp",
                "calendar_interval": "1h"
              },
              "aggs": {
                "error_rate": {
                  "filter": {"term": {"level": "error"}},
                  "aggs": {
                    "total": {"value_count": {"field": "level"}}
                  }
                }
              }
            }
          }
        }
      },
      "format": {"property": "aggregations.time_buckets.buckets"}
    },
    "mark": "area",
    "encoding": {
      "x": {
        "field": "key",
        "type": "temporal",
        "axis": {"title": "Time"}
      },
      "y": {
        "field": "error_rate.total.value",
        "type": "quantitative",
        "axis": {"title": "Error Count"}
      },
      "color": {"value": "#ff6b6b"}
    }
  }
}
```

#### TSVB (Time Series Visual Builder)
```json
{
  "tsvb_config": {
    "type": "timeseries",
    "series": [
      {
        "id": "series1",
        "metrics": [
          {
            "id": "metric1",
            "type": "avg",
            "field": "response_time"
          }
        ],
        "split_mode": "terms",
        "terms_field": "service.name",
        "terms_size": 10,
        "color": "#68BC00"
      }
    ],
    "time_field": "@timestamp",
    "interval": "5m",
    "axis_position": "left",
    "axis_formatter": "number",
    "axis_scale": "normal"
  }
}
```

## 실시간 모니터링

### 라이브 데이터 스트리밍
```json
{
  "dashboard_settings": {
    "refresh_interval": "10s",
    "time_range": {
      "from": "now-30m",
      "to": "now"
    },
    "auto_refresh": true,
    "real_time_mode": true
  },
  "panel_settings": {
    "streaming": {
      "enabled": true,
      "buffer_size": 1000,
      "update_frequency": "5s"
    }
  }
}
```

### 메트릭 추적 및 KPI 모니터링
```json
{
  "kpi_dashboard": {
    "panels": [
      {
        "type": "metric",
        "title": "System Health Score",
        "calculation": {
          "formula": "(success_rate * 0.4) + (performance_score * 0.3) + (availability * 0.3)",
          "fields": {
            "success_rate": "aggregation of non-error requests",
            "performance_score": "inverse of avg response time",
            "availability": "uptime percentage"
          }
        },
        "thresholds": {
          "critical": 70,
          "warning": 85,
          "good": 95
        }
      },
      {
        "type": "gauge",
        "title": "CPU Utilization",
        "field": "system.cpu.usage",
        "ranges": [
          {"from": 0, "to": 70, "color": "green"},
          {"from": 70, "to": 85, "color": "yellow"},
          {"from": 85, "to": 100, "color": "red"}
        ]
      }
    ]
  }
}
```

### 로그 패턴 분석 및 이상 감지
```json
{
  "anomaly_detection": {
    "jobs": [
      {
        "id": "response_time_anomaly",
        "description": "API response time anomaly detection",
        "analysis_config": {
          "bucket_span": "5m",
          "detectors": [
            {
              "function": "mean",
              "field_name": "response_time",
              "partition_field_name": "service.name"
            }
          ]
        },
        "data_description": {
          "time_field": "@timestamp",
          "time_format": "epoch_ms"
        }
      }
    ],
    "visualization": {
      "type": "ml_anomaly_explorer",
      "show_annotations": true,
      "severity_threshold": 75
    }
  }
}
```

### 사용자 정의 필드 계산
```json
{
  "scripted_fields": [
    {
      "name": "error_rate_percentage",
      "type": "number",
      "script": {
        "source": "doc['errors'].value / doc['total_requests'].value * 100"
      }
    },
    {
      "name": "response_time_category",
      "type": "string",
      "script": {
        "source": """
          def time = doc['response_time'].value;
          if (time < 100) return 'fast';
          else if (time < 500) return 'normal';
          else if (time < 2000) return 'slow';
          else return 'very_slow';
        """
      }
    }
  ]
}
```

## 대시보드 고급 기능

### 드릴다운 기능
```json
{
  "drilldown_config": {
    "panels": [
      {
        "id": "overview_panel",
        "drilldowns": [
          {
            "name": "Service Details",
            "target_dashboard": "service_detail_dashboard",
            "params": {
              "service_name": "{{event.value}}",
              "time_range": "{{time_range}}"
            }
          }
        ]
      }
    ]
  }
}
```

### 필터 연동 및 상호작용
```json
{
  "filter_interactions": {
    "global_filters": [
      {
        "field": "environment",
        "type": "dropdown",
        "options": ["production", "staging", "development"]
      }
    ],
    "panel_filters": [
      {
        "source_panel": "service_overview",
        "target_panels": ["error_details", "performance_metrics"],
        "filter_field": "service.name"
      }
    ]
  }
}
```

### 알림 통합
```json
{
  "dashboard_alerts": {
    "watchers": [
      {
        "name": "high_error_rate_alert",
        "condition": "error_rate > 5%",
        "actions": [
          {
            "type": "email",
            "to": ["ops-team@company.com"],
            "subject": "High Error Rate Detected"
          },
          {
            "type": "slack",
            "webhook": "https://hooks.slack.com/...",
            "channel": "#alerts"
          }
        ]
      }
    ]
  }
}
```

## 성능 최적화

### 대시보드 성능 튜닝
```json
{
  "performance_settings": {
    "panel_optimization": {
      "use_sampling": true,
      "sample_size": 10000,
      "cache_queries": true,
      "query_timeout": "30s"
    },
    "data_optimization": {
      "use_index_patterns": true,
      "limit_time_range": "7d",
      "minimize_field_usage": true
    }
  }
}
```

### 쿼리 최적화
```kql
# 효율적인 쿼리 작성
# 나쁜 예
*

# 좋은 예  
service: api AND @timestamp >= now-1h

# 필드 존재 여부 체크 최적화
# 나쁜 예
message: *

# 좋은 예
_exists_: message
```

## Best Practices

### 대시보드 설계 원칙
- **목적 중심 설계**: 명확한 비즈니스 목표에 따른 대시보드 구성
- **계층적 구조**: 개요 → 상세 → 드릴다운 순서로 정보 제공
- **시각적 일관성**: 색상, 폰트, 레이아웃의 일관된 적용
- **성능 고려**: 적절한 시간 범위와 데이터 샘플링 활용

### 사용자 경험 개선
- **반응형 설계**: 다양한 화면 크기에 대응
- **인터랙티브 요소**: 필터, 드릴다운, 툴팁 활용
- **로딩 성능**: 빠른 데이터 로딩과 진행 상태 표시
- **접근성**: 색각 이상자를 고려한 색상 선택

### 유지보수성
- **템플릿 활용**: 재사용 가능한 대시보드 템플릿 작성
- **문서화**: 대시보드 목적과 사용법 문서화
- **버전 관리**: 대시보드 설정의 버전 관리
- **정기 검토**: 대시보드 효용성 및 성능 정기 검토

## Benefits and Challenges

### Benefits
- **직관적 시각화**: 복잡한 데이터를 쉽게 이해할 수 있는 시각적 표현
- **실시간 모니터링**: 실시간 데이터 업데이트와 알림 기능
- **유연성**: 다양한 시각화 옵션과 커스터마이징 가능
- **협업 도구**: 팀 간 데이터 공유 및 협업 지원

### Challenges
- **성능 관리**: 복잡한 대시보드의 성능 최적화 필요
- **데이터 품질**: 시각화 품질이 원본 데이터 품질에 의존
- **사용자 교육**: 효과적인 대시보드 활용을 위한 사용자 교육 필요
- **유지보수**: 지속적인 대시보드 업데이트 및 관리 필요