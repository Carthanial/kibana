# Transactions

Transactions are stored in two different formats:

#### Individual transactions document

A single transaction event where `transaction.duration.us` is the latency.

```json
{
  "@timestamp": "2021-09-01T10:00:00.000Z",
  "processor.event": "transaction",
  "transaction.duration.us": 2000,
  "event.outcome": "success"
}
```

or

#### Aggregated (metric) document
A pre-aggregated document where `_doc_count` is the number of transaction events, and `transaction.duration.histogram` is the latency distribution. 

```json
{
  "_doc_count": 2,
  "@timestamp": "2021-09-01T10:00:00.000Z",
  "processor.event": "metric",
  "metricset.name": "transaction",
  "transaction.duration.histogram": {
    "counts": [1, 1],
    "values": [2000, 3000]
  },
  "event.outcome": "success"
}
```

The decision to use aggregated transactions or not is determined in [`getSearchAggregatedTransactions`](https://github.com/elastic/kibana/blob/a2ac439f56313b7a3fc4708f54a4deebf2615136/x-pack/plugins/apm/server/lib/helpers/aggregated_transactions/index.ts#L53-L79) and then used to specify [the transaction index](https://github.com/elastic/kibana/blob/a2ac439f56313b7a3fc4708f54a4deebf2615136/x-pack/plugins/apm/server/lib/suggestions/get_suggestions.ts#L30-L32) and [the latency field](https://github.com/elastic/kibana/blob/a2ac439f56313b7a3fc4708f54a4deebf2615136/x-pack/plugins/apm/server/lib/alerts/chart_preview/get_transaction_duration.ts#L62-L65)

### Latency

Latency is the duration of a transaction. This can be calculated using transaction events or metric events (aggregated transactions).

Noteworthy fields: `transaction.duration.us`, `transaction.duration.histogram`

#### Transaction-based latency

```json
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [{ "terms": { "processor.event": ["transaction"] } }]
    }
  },
  "aggs": {
    "latency": { "avg": { "field": "transaction.duration.us" } }
  }
}
```

#### Metric-based latency

```json
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "terms": { "processor.event": ["metric"] } },
        { "term": { "metricset.name": "transaction" } }
      ]
    }
  },
  "aggs": {
    "latency": { "avg": { "field": "transaction.duration.histogram" } }
  }
}
```

Please note: `metricset.name: transaction` was only recently introduced. To retain backwards compatability we still use the old filter `{ "exists": { "field": "transaction.duration.histogram" }}` when filtering for aggregated transactions ([see example](https://github.com/elastic/kibana/blob/2c8686770e64b82cf8e1db5a22327d40d5f8ce45/x-pack/plugins/apm/server/lib/helpers/aggregated_transactions/index.ts#L89-L95)).

### Throughput

Throughput is the number of transactions per minute. This can be calculated using transaction events or metric events (aggregated transactions).

Noteworthy fields: None (based on `doc_count`)

```js
{
  "size": 0,
  "query": {
    // same filters as for latency
  },
  "aggs": {
    "throughput": { "rate": { "unit": "minute" } }
  }
}
```

### Failed transaction rate

Failed transaction rate is the number of transactions with `event.outcome=failure` per minute.
Noteworthy fields: `event.outcome`

```js
{
  "size": 0,
  "query": {
    // same filters as for latency
  },
  "aggs": {
    "outcomes": {
      "terms": {
        "field": "event.outcome",
        "include": ["failure", "success"]
      }
    }
  }
}
```

# System metrics

System metrics are captured periodically (every 60 seconds by default).

### CPU

![image](https://user-images.githubusercontent.com/209966/135990500-f85bd8d9-b5a5-4b7c-b9e1-0759eefb8a29.png)

Used in: [Metrics section](https://github.com/elastic/kibana/blob/00bb59713ed115343eb70d4e39059476edafbade/x-pack/plugins/apm/server/lib/metrics/by_agent/shared/cpu/index.ts#L83)

Noteworthy fields: `system.cpu.total.norm.pct`, `system.process.cpu.total.norm.pct`

#### Sample document

```json
{
  "@timestamp": "2021-09-01T10:00:00.000Z",
  "processor.event": "metric",
  "metricset.name": "app",
  "system.process.cpu.total.norm.pct": 0.003,
  "system.cpu.total.norm.pct": 0.28
}
```

#### Query

```json
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "terms": { "processor.event": ["metric"] } },
        { "terms": { "metricset.name": ["app"] } }
      ]
    }
  },
  "aggs": {
    "systemCPUAverage": { "avg": { "field": "system.cpu.total.norm.pct" } },
    "processCPUAverage": {
      "avg": { "field": "system.process.cpu.total.norm.pct" }
    }
  }
}
```

### Memory

![image](https://user-images.githubusercontent.com/209966/135990556-31716990-2812-46c3-a926-8c2a64c7c89f.png)

Noteworthy fields: `system.memory.actual.free`, `system.memory.total`,

#### Sample document

```json
{
  "@timestamp": "2021-09-01T10:00:00.000Z",
  "processor.event": "metric",
  "metricset.name": "app",
  "system.memory.actual.free": 13182939136,
  "system.memory.total": 15735697408
}
```

#### Query

```js
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "terms": { "processor.event": ["metric"] }},
        { "terms": { "metricset.name": ["app"] }}

        // ensure the memory fields exists
        { "exists": { "field": "system.memory.actual.free" }},
        { "exists": { "field": "system.memory.total" }},
      ]
    }
  },
  "aggs": {
    "memoryUsedAvg": {
      "avg": {
        "script": {
          "lang": "expression",
          "source": "1 - doc['system.memory.actual.free'] / doc['system.memory.total']"
        }
      }
    }
  }
}
```

Above example is overly simplified. In reality [we do a bit more](https://github.com/elastic/kibana/blob/fe9b5332e157fd456f81aecfd4ffa78d9e511a66/x-pack/plugins/apm/server/lib/metrics/by_agent/shared/memory/index.ts#L51-L71) to properly calculate memory usage inside containers



# Transaction breakdown metrics (`transaction_breakdown`)

A pre-aggregations of transaction documents where `transaction.breakdown.count` is the number of original transactions.

Noteworthy fields: `transaction.name`, `transaction.type`

#### Sample document

```json
{
  "@timestamp": "2021-09-27T21:59:59.828Z",
  "processor.event": "metric",
  "metricset.name": "transaction_breakdown",
  "transaction.breakdown.count": 12,
  "transaction.name": "GET /api/products",
  "transaction.type": "request"
}
}
```

# Span breakdown metrics (`span_breakdown`)

A pre-aggregations of span documents where `span.self_time.count` is the number of original spans. Measures the "self-time" for a span type, and optional subtype, within a transaction group. 

Span breakdown metrics are used to power the "Time spent by span type" graph. Agents collect summarized metrics about the timings of spans, broken down by `span.type`.

![image](https://user-images.githubusercontent.com/209966/135990865-9077ae3e-a7a4-4b5d-bdce-41dc832689ea.png)

Used in: ["Time spent by span type" chart](https://github.com/elastic/kibana/blob/723370ab23573e50b3524a62c6b9998f2042423d/x-pack/plugins/apm/server/lib/transactions/breakdown/index.ts#L48-L87)

Noteworthy fields: `transaction.name`, `transaction.type`, `span.type`, `span.subtype`, `span.self_time.*`

#### Sample document

```json
{
  "@timestamp": "2021-09-27T21:59:59.828Z",
  "processor.event": "metric",
  "metricset.name": "span_breakdown",
  "transaction.name": "GET /api/products",
  "transaction.type": "request",
  "span.self_time.sum.us": 1028,
  "span.self_time.count": 12,
  "span.type": "db",
  "span.subtype": "elasticsearch"
}
```

#### Query

```json
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "terms": { "processor.event": ["metric"] } },
        { "terms": { "metricset.name": ["span_breakdown"] } }
      ]
    }
  },
  "aggs": {
    "total_self_time": { "sum": { "field": "span.self_time.sum.us" } },
    "types": {
      "terms": { "field": "span.type" },
      "aggs": {
        "subtypes": {
          "terms": { "field": "span.subtype" },
          "aggs": {
            "self_time_per_subtype": {
              "sum": { "field": "span.self_time.sum.us" }
            }
          }
        }
      }
    }
  }
}
```

# Service destination metrics

Pre-aggregations of span documents, where `span.destination.service.response_time.count` is the number of original spans.
These metrics measure the count and total duration of requests from one service to another service.

![image](https://user-images.githubusercontent.com/209966/135990117-170070da-2fc5-4014-a597-0dda0970854c.png)

Used in: [Dependencies (latency)](https://github.com/elastic/kibana/blob/00bb59713ed115343eb70d4e39059476edafbade/x-pack/plugins/apm/server/lib/backends/get_latency_charts_for_backend.ts#L68-L79), [Dependencies (throughput)](https://github.com/elastic/kibana/blob/00bb59713ed115343eb70d4e39059476edafbade/x-pack/plugins/apm/server/lib/backends/get_throughput_charts_for_backend.ts#L67-L74) and [Service Map](https://github.com/elastic/kibana/blob/00bb59713ed115343eb70d4e39059476edafbade/x-pack/plugins/apm/server/lib/service_map/get_service_map_backend_node_info.ts#L57-L67)

Noteworthy fields: `span.destination.service.*`

#### Sample document

A pre-aggregated document with 73 span requests from opbeans-ruby to elasticsearch, and a combined latency of 1554ms

```json
{
  "@timestamp": "2021-09-01T10:00:00.000Z",
  "processor.event": "metric",
  "metricset.name": "service_destination",
  "service.name": "opbeans-ruby",
  "span.destination.service.response_time.count": 73,
  "span.destination.service.response_time.sum.us": 1554192,
  "span.destination.service.resource": "elasticsearch",
  "event.outcome": "success"
}
```

### Latency

The latency between a service and an (external) endpoint

```json
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "terms": { "processor.event": ["metric"] } },
        { "term": { "metricset.name": "service_destination" } },
        { "term": { "span.destination.service.resource": "elasticsearch" } }
      ]
    }
  },
  "aggs": {
    "latency_sum": {
      "sum": { "field": "span.destination.service.response_time.sum.us" }
    },
    "latency_count": {
      "sum": { "field": "span.destination.service.response_time.count" }
    }
  }
}
```

### Throughput

Captures the number of requests made from a service to an (external) endpoint


#### Query

```json
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "terms": { "processor.event": ["metric"] } },
        { "term": { "metricset.name": "service_destination" } },
        { "term": { "span.destination.service.resource": "elasticsearch" } }
      ]
    }
  },
  "aggs": {
    "throughput": {
      "rate": {
        "field": "span.destination.service.response_time.count",
        "unit": "minute"
      }
    }
  }
}
```

## Common filters

Most Elasticsearch queries will need to have one or more filters. There are a couple of reasons for adding filters:

- correctness: Running an aggregation on unrelated documents will produce incorrect results
- stability: Running an aggregation on unrelated documents could cause the entire query to fail
- performance: limiting the number of documents will make the query faster

```js
{
  "query": {
    "bool": {
      "filter": [
        // service name
        { "term": { "service.name": "opbeans-go" }},

        // service environment
        { "term": { "service.environment": "testing" }}

        // transaction type
        { "term": { "transaction.type": "request" }}

        // event type (possible values : transaction, span, metric, error)
        { "terms": { "processor.event": ["metric"] }},

        // metric set is a subtype of `processor.event: metric`
        { "terms": { "metricset.name": ["transaction"] }},

        // time range
        {
          "range": {
            "@timestamp": {
              "gte": 1633000560000,
              "lte": 1633001498988,
              "format": "epoch_millis"
            }
          }
        }
      ]
    }
  },
```
