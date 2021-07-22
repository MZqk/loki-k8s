# loki-k8s

部署

微服务日志下载API：

已服务名称和时间关联下载

## `GET /loki/api/v1/query_range`

例子：

/loki/api/v1/query_range?direction=BACKWARD&limit=1000&query={container="nginx"}&start=16234806200000000&end=1626076221000000000&step=1

`/loki/api/v1/query_range` 用于在一段时间内进行查询，并接受 URL 中的以下查询参数：

- `query`：要执行的[LogQL](https://grafana.com/docs/loki/latest/logql/)查询
- `limit`: 要返回的最大条目数
- `start`: 查询的开始时间，以Unix 时间戳表示。默认为一小时前。
- `end`: 查询的结束时间，以Unix 时间戳表示。默认为现在。
- `step`: 以`duration`格式或浮点秒数查询分辨率步长。`duration`指形式为 的 Prometheus 持续时间字符串`[0-9]+[smhdwy]`。例如，5m 表示持续时间为 5 分钟。默认为基于`start`和的动态值`end`。仅适用于产生矩阵响应的查询类型。
- `interval`：**实验性，见下文**仅返回（或大于）指定间隔的条目，可以是`duration`格式或浮点数秒。仅适用于产生流响应的查询。
- `direction`：确定日志的排序顺序。支持的值为`forward`或`backward`。默认为`backward.`

在微服务模式下，`/loki/api/v1/query_range`由查询器和前端公开。

##### 步骤与间隔

在`step`对 Loki 进行指标查询或返回矩阵响应的查询时使用该参数。它的计算方式与 Prometheus 的计算方式完全相同`step`。首先查询将被评估，`start`然后再次评估 at`start + step`和再次 at`start + step + step`直到`end`到达。结果将是在每一步评估的查询结果的矩阵。

在`interval`对 Loki 进行日志查询或返回流响应的查询时使用该参数。它通过返回一个日志条目 at 进行评估`start`，然后下一个条目将返回一个时间戳 >= 的条目`start + interval`，然后再次 at`start + interval + interval`等等，直到`end`到达。它不会填充缺失的条目。

**请注意 interval 的实验性质**此标志将来可能会被删除，如果是这样，它可能会支持 LogQL 表达式来执行类似的行为，但目前尚不确定。 创建[问题 1779](https://github.com/grafana/loki/issues/1779)是为了跟踪讨论，如果您正在使用，`interval`请将您的用例和想法添加到该问题。

回复：

```
{
  "status": "success",
  "data": {
    "resultType": "matrix" | "streams",
    "result": [<matrix value>] | [<stream value>]
    "stats" : [<statistics>]
  }
}
```

在哪里`<matrix value>`：

```
{
  "metric": {
    <label key-value pairs>
  },
  "values": [
    <number: second unix epoch>,
    <string: value>
  ]
}
```

并且`<stream value>`是：

```
{
  "stream": {
    <label key-value pairs>
  },
  "values": [
    [
      <string: nanosecond unix epoch>,
      <string: log line>
    ],
    ...
  ]
}
```

有关Loki 返回的统计信息的信息，请参阅[统计](https://grafana.com/docs/loki/latest/api/#Statistics)信息。

### 例子

```bash
$ curl -G -s  "http://localhost:3100/loki/api/v1/query_range" --data-urlencode 'query=sum(rate({job="varlogs"}[10m])) by (level)' --data-urlencode 'step=300' | jq
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
       "metric": {
          "level": "info"
        },
        "values": [
          [
            1588889221,
            "137.95"
          ],
          [
            1588889221,
            "467.115"
          ],
          [
            1588889221,
            "658.8516666666667"
          ]
        ]
      },
      {
        "metric": {
          "level": "warn"
        },
        "values": [
          [
            1588889221,
            "137.27833333333334"
          ],
          [
            1588889221,
            "467.69"
          ],
          [
            1588889221,
            "660.6933333333334"
          ]
        ]
      }
    ],
    "stats": {
      ...
    }
  }
}
```
