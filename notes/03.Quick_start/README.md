## Quick start

这里使用 `Kibana` 的开发工具控制台向 `es` 提交 REST 请求。

### Add data to Elasticsearch

发送带有 JSON 文档的 HTTP post 请求；

```
POST /customer/_doc/1
{
  "name": "John Doe"
}
```

该请求会自动创建索引 `customer`，加入拥有 ID 为 1 的新文档并所以字段 `name`；

*在阿里云 7.10.0 版本中默认设置为不能自动创建索引。修改配置后，需要重启，并且需要很久才能生效；*

新的文档立即生效；通过一个 GET 请求能过获取指定 ID 的文档；

```
GET /customer/_doc/1
```

### Add data in bulk

使用 `_bulk` 接口可以一次添加多个文档；

最佳的批处理大小取决于：文档大小、复杂性、索引和索引负载，以及集群可用的资源；可以考虑一次性提交 1000~5000 个文档并且大小控制在 5MB~15MB。

**批量数据必须使用每行一个 JSON 的格式（newline-delimited JSON，NDJSON）。每行结尾必须使用一个换行符（\n），包括最后一行。**

以下代码向 `bank` 索引添加 10 条文档；

```
PUT bank/_bulk
{ "create":{ } }
{ "account_number":1,"balance":39225,"firstname":"Amber","lastname":"Duke","age":32,"gender":"M","address":"880 Holmes Lane","employer":"Pyrami","email":"amberduke@pyrami.com","city":"Brogan","state":"IL" }
{ "create":{ } }
{ "account_number":6,"balance":5686,"firstname":"Hattie","lastname":"Bond","age":36,"gender":"M","address":"671 Bristol Street","employer":"Netagy","email":"hattiebond@netagy.com","city":"Dante","state":"TN" }
{ "create":{ } }
{ "account_number":13,"balance":32838,"firstname":"Nanette","lastname":"Bates","age":28,"gender":"F","address":"789 Madison Street","employer":"Quility","email":"nanettebates@quility.com","city":"Nogal","state":"VA" }
{ "create":{ } }
{ "account_number":18,"balance":4180,"firstname":"Dale","lastname":"Adams","age":33,"gender":"M","address":"467 Hutchinson Court","employer":"Boink","email":"daleadams@boink.com","city":"Orick","state":"MD" }
{ "create":{ } }
{ "account_number":20,"balance":16418,"firstname":"Elinor","lastname":"Ratliff","age":36,"gender":"M","address":"282 Kings Place","employer":"Scentric","email":"elinorratliff@scentric.com","city":"Ribera","state":"WA" }
{ "create":{ } }
{ "account_number":25,"balance":40540,"firstname":"Virginia","lastname":"Ayala","age":39,"gender":"F","address":"171 Putnam Avenue","employer":"Filodyne","email":"virginiaayala@filodyne.com","city":"Nicholson","state":"PA" }
{ "create":{ } }
{ "account_number":32,"balance":48086,"firstname":"Dillard","lastname":"Mcpherson","age":34,"gender":"F","address":"702 Quentin Street","employer":"Quailcom","email":"dillardmcpherson@quailcom.com","city":"Veguita","state":"IN" }
{ "create":{ } }
{ "account_number":37,"balance":18612,"firstname":"Mcgee","lastname":"Mooney","age":39,"gender":"M","address":"826 Fillmore Place","employer":"Reversus","email":"mcgeemooney@reversus.com","city":"Tooleville","state":"OK" }
{ "create":{ } }
{ "account_number":44,"balance":34487,"firstname":"Aurelia","lastname":"Harding","age":37,"gender":"M","address":"502 Baycliff Terrace","employer":"Orbalix","email":"aureliaharding@orbalix.com","city":"Yardville","state":"DE" }
{ "create":{ } }
{ "account_number":49,"balance":29104,"firstname":"Fulton","lastname":"Holt","age":23,"gender":"F","address":"451 Humboldt Street","employer":"Anocha","email":"fultonholt@anocha.com","city":"Sunriver","state":"RI" }

```

*批处理的服务器会返回每一条文档的处理结果，每条中拥有 REST 的请求结果例如 `201`。其中也会列出分片的数量，以及成功提交的分片的数量。本次批处理的数据的 ID 是自动生成。*

### Search and sort data

在某一个字段中搜索，需要使用 `match` 查询；一下请求搜索 `address` 字段，找到地址中包含 *mill* 或 *lane* 的顾客；

```
GET /bank/_search
{
  "query": {"match": {"address": "mill lane"}}
}
```

要构建更复杂的查询，可以使用 bool 查询来组合多个查询条件。根据需要（必须匹配）、想要（应该匹配）或不需要（必须不匹配）来设定条件；

以下请求查询年龄在 39 岁并且不住在宾夕法尼亚州（PA）的顾客；

```
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "39" } }
      ],
      "must_not": [
        { "match": { "state": "PA" } }
      ]
    }
  }
}
```

### Search and explore your data with Discover

除了发送 REST 请求，还可以使用 Discover 搜索、过滤数据、获取字段的结构信息或者展示搜索结果。

Kibana 需要先创建索引模式（data view），索引模式可以指向一个或多个索引，数据流或索引别名。

### Visualize your data

Kibana 中可以创建可视化和仪表板。

