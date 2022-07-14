## Query DSL

`es` 提供了一个完整的基于 JSON 的查询领域特定语言（Domain specific Language，DSL）来定义查询。可以把它看作查询的抽象语法树（Abstract syntax Tree，AST），由两种类型的子句组成：

- 叶子查询子句

  叶子查询子句在指定字段中查找特定值，例如 `match`、`term` 和 `range` 查询。

- 复合查询子句

  复合查询子句包含其它叶子查询或复合查询，用于以逻辑方式组合多个查询（如 `bool` 或者 `dis_max` 查询），或更改它们的行为（如 `constant_score` 查询）。

查询子句的行为取决于它们是在查询上下文中还是在筛选上下文中使用。

**Allow expensive queries**

由于执行查询方式的不同，某些类型的查询通常执行缓慢，这可能会影响集群的稳定性。这些查询可以分类如下：

- 需要进行线性扫面来识别匹配得查询：
  - `script` 查询
  - 数值、日期、布尔、ip、地理点或者关键词（keyword）字段没有被索引但是开启了文档值
- 前置成本较高得查询：
  - `fuzzy` 查询（除了 `wildcard` 字段）
  - `regexp` 查询（除了 `wildcard` 字段）
  - `prefix` 查询（除了 `wildcard` 字段和没有 `index_prefixes` 的字段）
  - 在 `text` 和 `keyword` 上的 `range` 查询
- 关联（Joining）查询
- 查询的每个文档成本可能很高：
  - `script_score` 查询
  - `percolate` 查询

可以将 `search.allow_expensive_queries` 设置为 `false` 来避免这类查询的执行（默认为 `true`）。
