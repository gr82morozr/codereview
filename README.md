{
  "size": 0,
  "aggs": {
    "min_ts": { "min": { "field": "load_timestamp" } },
    "max_ts": { "max": { "field": "load_timestamp" } },
    "total_docs": { "value_count": { "field": "load_timestamp" } }
  }
}
