# Elasticsearch使用

## 使用script对日期字段进行按小时聚合
这里使用expression脚本对字段进行聚合，要求字段必须是date类型；
expression脚本只能对numberic、date、geo_point这三种字段类型使用
```json
{
  "from": 0,
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        {
          "bool": {
            "must": [
              {
                "range": {
                  "pass_time": {
                    "from": "2000-01-01 16:12:15",
                    "to": "2022-06-24 16:12:15",
                    "include_lower": true,
                    "include_upper": true,
                    "boost": 1.0
                  }
                }
              }
            ],
            "adjust_pure_negative": true,
            "boost": 1.0
          }
        }
      ],
      "adjust_pure_negative": true,
      "boost": 1.0
    }
  },
  "_source": {
    "includes": [
      "pass_time"
    ],
    "excludes": []
  },
  "stored_fields": "pass_time",
  "aggregations": {
    "pass_time.keyword": {
      "terms": {
        "size":100,
        "script": {
          "lang": "expression",
          "source": "doc['pass_time'].date.hourOfDay"
        }
      }
    }
  }
}}
```

## 使用script对文本类型的日期内容字段进行按小时聚合
这里使用painless脚本的字符串截取来操作
```json
{
  "from": 0,
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        {
          "bool": {
            "must": [
              {
                "range": {
                  "pass_time.keyword": {
                    "from": "2000-01-01 16:12:15",
                    "to": "2022-06-24 16:12:15",
                    "include_lower": true,
                    "include_upper": true,
                    "boost": 1.0
                  }
                }
              }
            ],
            "adjust_pure_negative": true,
            "boost": 1.0
          }
        }
      ],
      "adjust_pure_negative": true,
      "boost": 1.0
    }
  },
  "_source": {
    "includes": [
      "pass_time"
    ],
    "excludes": []
  },
  "stored_fields": "pass_time",
  "aggregations": {
    "pass_time.keyword": {
      "terms": {
        "size":100,
        "script": {
          "source": "doc['pass_time.keyword'].value.substring(11,13)"
        }
      }
    }
  }
}}
```