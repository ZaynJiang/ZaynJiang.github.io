## DSL

```sql

GET /flink_span_minute_record_202311*/_search
{
  "size": 1,
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "appName": "yunxin_22_50"
          }
        },
        {
          "term": {
            "spanLayer": {
              "value": "Database"
            }
          }
        },
        {
          "range": {
            "timestamp": {
              "gte": 1700029842000
            }
          }
        },
                {
                    "range": {
                        "avgDuration": {
                            "gte": 200
                        }
                    }
                }
      ]
    }
  },
  "aggs": {
    "执行次数区间": {
      "range": {
        "field": "count",
        "ranges": [
          {
            "to": 100
          },
          {
            "from": 100,
            "to": 200
          },
          {
            "from": 200
          }
        ]
      },
      "aggs": {
        "指标": {
          "terms": {
            "field": "item",
            "size": 20,
            "order": {
              "最大耗时": "desc"
            }
          },
          "aggs": {
            "平均耗时": {
              "avg": {
                "field": "avgDuration"
              }
            },
            "最大耗时": {
              "max": {
                "field": "avgDuration"
              }
            },
            "sql次数": {
              "sum": {
                "field": "count"
              }
            },
            "失败次数": {
              "sum": {
                "field": "failNum"
              }
            },
            "请求总耗时": {
              "sum": {
                "field": "sumDuration"
              }
            }
          }
        }
      }
    }
  }
}
```

