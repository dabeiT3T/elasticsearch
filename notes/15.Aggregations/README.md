## Aggregations

聚合以维度、统计或者其它分析方式概述数据。聚合回答例如这样的问题：

- 网页的平均加载时间是多少？
- 根据交易量，谁是最有价值的客户？
- 在网络中，什么文件被认为是大文件？
- 每个产品目录下有多少产品？

`es` 将聚合组织为三个目录：

- Metric aggregations

  计算维度，例如字段值的总和或者平均值。

- Bucket aggregations

  将文档组合到桶，也称为箱，基于字段值、范围或者其它维度。

- Pipeline aggregations

  将其它聚合作为输入，而不是文档或者字段。

### Run an aggregation

通过指定搜索 API 的 `aggs` 参数，作为搜索的一部分执行聚合。下方搜索在 `my-field` 字段上执行项聚合（terms aggregation）：

```
GET /my-index-000001/_search
{
  "aggs": {
    "my-agg-name": {
      "terms": {
        "field": "my-field"
      }
    }
  }
}
```

聚合结果在响应的 `aggregations` 对象中：

```json
{
  "took": 78,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 5,
      "relation": "eq"
    },
    "max_score": 1.0,
    "hits": []
  },
  "aggregations": {
    "my-agg-name": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": []
    }
  }
}
```

1. `my-agg-name` 是 `my-agg-name` 的聚合结果。

### Change an aggregation's scope

使用 `query` 参数限制聚合运行的文档：

```
GET /my-index-000001/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-1d/d",
        "lt": "now/d"
      }
    }
  },
  "aggs": {
    "my-agg-name": {
      "terms": {
        "field": "my-field"
      }
    }
  }
}
```

### Return only aggregation results

默认，搜索包含聚合时，返回搜索匹配和聚合结果。想要只包含聚合结果，将 `size` 设置为 `0`：

```
GET /my-index-000001/_search
{
  "size": 0,
  "aggs": {
    "my-agg-name": {
      "terms": {
        "field": "my-field"
      }
    }
  }
}
```

### Run multiple aggregations

在同一个请求中可以指定多个聚合：

```
GET /my-index-000001/_search
{
  "aggs": {
    "my-first-agg-name": {
      "terms": {
        "field": "my-field"
      }
    },
    "my-second-agg-name": {
      "avg": {
        "field": "my-other-field"
      }
    }
  }
}
```

### Run sub-aggregations

桶聚合支持桶或者维度的子聚合。例如，一个项聚合（terms aggregation）用一个平均（avg）子聚合计算每一个文档桶的平均值。内嵌子聚合的级数或深度没有限制。

```
GET /my-index-000001/_search
{
  "aggs": {
    "my-agg-name": {
      "terms": {
        "field": "my-field"
      },
      "aggs": {
        "my-sub-agg-name": {
          "avg": {
            "field": "my-other-field"
          }
        }
      }
    }
  }
}
```

响应的内嵌子聚合结果在其父聚合之下：

```json
{
  "aggregations": {
    "my-agg-name": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "foo",
          "doc_count": 5,
          "my-sub-agg-name": {
            "value": 75.0
          }
        }
      ]
    }
  }
}
```

1. `my-agg-name` 是父聚合的结果。
2. `my-sub-agg-name` 是 `my-agg-name` 的子聚合的结果。

### Add custom metadata

使用 `meta` 对象将自定义元数据与聚合关联：

```
GET /my-index-000001/_search
{
  "aggs": {
    "my-agg-name": {
      "terms": {
        "field": "my-field"
      },
      "meta": {
        "my-metadata-field": "foo"
      }
    }
  }
}
```

响应中在适当位置返回 `meta` 对象：

```json
{
  "aggregations": {
    "my-agg-name": {
      "meta": {
        "my-metadata-field": "foo"
      },
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": []
    }
  }
}
```

### Return the aggregation type

默认，聚合结果只包含聚合的名字，而不是其聚合的类型。要返回聚合的类型，使用 `typed_keys` 查询参数。

```
GET /my-index-000001/_search?typed_keys
{
  "aggs": {
    "my-agg-name": {
      "histogram": {
        "field": "my-field",
        "interval": 1000
      }
    }
  }
}
```

响应返回聚合的类型作为聚合名字的前缀。

> 有些聚合返回不同于请求中聚合的类型。例如，`terms`、`significant terms` 和 `percentiles` 聚合基于聚合字段的数据类型返回聚合类型。

```
{
  ...
  "aggregations": {
    "histogram#my-agg-name": {
      "buckets": []
    }
  }
}
```

1. 聚合类型 `histogram` 紧跟 `#` 分隔符和聚合名字 `my-agg-name`。

### Use scripts in an aggregation

当字段不能完全匹配所需的聚合时，可以在一个运行时字段（runtime field）上进行聚合：

```
GET /my-index-000001/_search?size=0
{
  "runtime_mappings": {
    "message.length": {
      "type": "long",
      "script": "emit(doc[\u0027message.keyword\u0027].value.length())"
    }
  },
  "aggs": {
    "message_length": {
      "histogram": {
        "interval": 10,
        "field": "message.length"
      }
    }
  }
}
```

脚本动态地计算字段值，这给聚合增加了一点点开销。除了花费在计算上地时间之外，一些例如 `terms` 和 `filters` 聚合不能对运行时字段使用它们地一些优化。总的来说，使用运行时字段地性能成本因聚合而异。

### Aggregation caches

为了更快的响应，`es` 将频繁运行的聚合的结果缓存在分片请求缓存（shard request cache）中。要获取缓存的结果，每个请求使用相同的 `preference` 字符串。如果无需搜索结果，将 `size` 设置为 `0` 以避免填充为缓存。

`es` 将相同的偏好（preference）字符串的搜索路路由到同一个分片（shard）上。如果分片的数据在搜索之间没有修改，则分片返回缓存了的聚合结果。

### Limits for `long` values

当运行聚合时，`es` 使用 `double` 值存储和展示数字类型数据。结果就是，在 `long` 类型上数字大于 `2^53` 的聚合是近似值。
