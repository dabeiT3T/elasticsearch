## Search your data

### Run a search

查询接口接受 Query DSL 格式的查询内容。

以下请求使用 `match` 搜索索引 `my-index-000001`中 `user.id` 值是 `kimchy` 的文档。

```
GET /my-index-000001/_search
{
  "query": {
    "match": {
      "user.id": "kimchy"
    }
  }
}
```

接口在 `hits.hits` 中返回 10 条最匹配的文。

```json
{
  "took": 5,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 1.3862942,
    "hits": [
      {
        "_index": "my-index-000001",
        "_id": "kxWFcnMByiguvud1Z8vC",
        "_score": 1.3862942,
        "_source": {
          "@timestamp": "2099-11-15T14:12:12",
          "http": {
            "request": {
              "method": "get"
            },
            "response": {
              "bytes": 1070000,
              "status_code": 200
            },
            "version": "1.1"
          },
          "message": "GET /search HTTP/1.1 200 1070000",
          "source": {
            "ip": "127.0.0.1"
          },
          "user": {
            "id": "kimchy"
          }
        }
      }
    ]
  }
}
```

### Define fields that exist only in a query

*该功能 7.10.0 不包含。*

可以在查询中指定 `runtime_mappings` 参数，定义只在运行时生成的字段。

以下定义了一个运行时字段 `day_of_week`。脚本基于 `@timestamp` 字段，计算星期几。

同时还包含了一个操作 `day_of_week` 的桶聚合（terms aggregation）。

```
GET /my-index-000001/_search
{
  "runtime_mappings": {
    "day_of_week": {
      "type": "keyword",
      "script": {
        "source":
        "emit(doc[\u0027@timestamp\u0027].value.dayOfWeekEnum\n        .getDisplayName(TextStyle.FULL, Locale.ROOT))"
      }
    }
  },
  "aggs": {
    "day_of_week": {
      "terms": {
        "field": "day_of_week"
      }
    }
  }
}
```

响应包含基于运行时字段 `day_of_week` 的聚合。在 `buckets` 下，`key` 的值是 `Sunday`。查询动态计算运行时字段，就算这个字段没有被索引。

```json
{
  // ...
  // ***
  "aggregations" : {
    "day_of_week" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "Sunday",
          "doc_count" : 5
        }
      ]
    }
  }
}
```

### Common search options

#### Query DSL

可以混合多种查询类型，包含：

- 布尔值和其他复合查询，根据多个条件组合查询和匹配结果；
- 术语级别（Term-level）查询可以过滤和精确查找；
- 全文索引；
- 地理和空间查询；

#### Aggregations

##### Search multiple data streams and indices

使用逗号分隔或者类 grep 索引模式在一个请求中查找多个数据流和索引。甚至可以从特定的索引提高搜索结果。

##### Paginate search results

默认返回 10 条最佳匹配。可以获取更多或者更少。

##### Retrieve selected fields

响应的 `hits.hits` 包含了每个匹配完整的文档 `_source`。可以检索 `_source` 的子集或者其它字段。

##### Sort search results

默认搜索结果按照 `_score` 匹配度分数排序。使用 `script_score` 查询自定义分数的计算，或者使用其它字段排序。

#### Run an async search

`es` 默认是同步搜索的，当搜索结果完成后返回响应结果。

大数据或者多集群可能会花费很久时间，这时可以使用异步搜索。异步搜索可以检索部分结果并在以后获得完整结果。

### Search timeout

默认没有超时，请求等待所有分片完成搜索。

虽然同步搜索是为长时间搜索设计的，但是也可以使用 `timeout` 参数指定每个分片运行的时长。如果搜索时长到了，而 `es` 没有完成搜索，`es` 将已经搜出的整合为结果。一个搜索请求的总延迟取决于搜索所需要的分片数量和并发分片请求的数量。

使用集群设置 API（cluster setting API）中 `search.default_search_timeout` 参数，设置集群范围内所有请求的超时时间。如果请求中没有设置 `timeout` 参数，那么全局超时将作为超时的时长。如果搜索完成前，全局搜索时长超时了，请求将被任务清除（task cancellation）清除。默认 `search.default_search_timeout` 被设置为 -1（没有超时）。

```
GET /my-index-000001/_search
{
  "timeout": "2s",
  "query": {
    "match": {
      "user.id": "kimchy"
    }
  }
}
```

### Search cancellation

你可以使用任务管理 API（task management API）来取消一个搜索请求。`es` 同样会在客户端的 HTTP 连接关闭时，自动取消搜索请求。当搜索请求中止或超时时，推荐通过设置客户端来关闭 HTTP 连接。

### Track total hits

通常，在不访问所有匹配项的情况下，无法准确地计算总的命中计数，这对于匹配大量文档的查询来说代价很高。`track_total_hits` 参数可以控制命中数是如果被跟踪的。考虑到通常有个命中下限次数就够了，比如 ”至少命中 10000 次“，默认值为 `10000`。如果到达某个阈值后不需要准确的命中数，那么这是一个加快搜索速度的一个很好的折衷方案。

当 `track_total_hits` 设置为 `true`，搜索结果将总是跟踪匹配搜索结果的精准命中数（此时 `total.relation` 等于 `eq`）。否则，`total.relation` 返回搜索结果中 `total` 对象的含义。值 `gte` 代表 `total.value` 小于真实匹配的数量，值 `eq` 表示 `total.value` 就是精确的命中数。

```
GET /my-index-000001/_search
{
  "track_total_hits": true,
  "query": {
    "match" : {
      "user.id" : "elkbee"
    }
  }
}
```

返回

```json
{
  "_shards": {
    // ...
  },
  "timed_out": false,
  "took": 100,
  "hits": {
    "max_score": 1.0,
    "total" : {
      "value": 2048,	// 42 个文档匹配
      "relation": "eq"  // 计数是精确的
    },
    "hits": [
      // ...
    ]
  }
}
```

`track_total_hits` 可以设置为整数。以下代码将精确跟踪需要匹配的 100 个文档的命中数。

```
GET /my-index-000001/_search
{
  "track_total_hits": 100,
  "query": {
    "match": {
      "user.id": "elkbee"
    }
  }
}
```

响应中的 `hits.total.relation` 表示 `hits.total.value` 是一个精确值（`eq`）还是一个小于总数的下界（`gte`）。

```json
{
  "_shards": {
    // ...
  },
  "timed_out": false,
  "took": 30,
  "hits": {
    "max_score": 1.0,
    "total": {
      "value": 100,		// 至少有 100 个文档匹配
      "relation": "gte"	// 这是一个下界
    },
    "hits": [
        // ...
    ]
  }
}
```

如果完全不需要跟踪总的命中数，可以将 `track_total_hits` 设置为 `false` 来优化搜索时间。

```
GET /my-index-000001/_search
{
  "track_total_hits": false,
  "query": {
    "match": {
      "user.id": "elkbee"
    }
  }
}
```

返回

```json
{
  "_shards": {
      // ...
  }
  "timed_out": false,
  "took": 10,
  "hits": {				// 命中总数未知
    "max_score": 1.0,
    "hits": [
      // ...
    ]
  }
}
```

### Quickly check for matching docs

如果只关心是否有匹配项，可以将 `size` 设置为 0 表示对于搜索结果没有兴趣。也可以将 `terminate_after` 设置为 1，表示当有第一个匹配文档被发现时（每个分片），查询会被终止。

```
GET /_search?q=user.id:elkbee&size=0&terminate_after=1
```

> `terminate_after` 在 `post_filter` 之后生效，停止搜索；就像当分片中足够的命中数收集之后，聚合才执行。在聚合上的文档统计可能无法显示出响应的 `hits.total`，因为聚合在后筛选前执行。

由于 `size` 设置为 0，响应不会包含命中数据。`hits.total` 如果是 0，表示没有匹配的文档；或者大于 0 意味着在提前终止时，至少有相同数量的文档匹配查询。如果查询被提前终止，`terminated_early` 等于 `true`。

```json
{
  "took": 3,
  "timed_out": false,
  "terminated_early": true,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped" : 0,
    "failed": 0
  },
  "hits": {
    "total" : {
        "value": 1,
        "relation": "eq"
    },
    "max_score": null,
    "hits": []
  }
}
```

响应中 `took` 包含了请求处理所花费的毫秒时间，从节点收到查询开始，直到完成所有与搜索工作相关的工作并将上述的 JSON 返回给客户端之前。这意味着它包含在线程池中等待的时间，跨整个集群执行分布式搜索和收集所有结果所花费的时间。
