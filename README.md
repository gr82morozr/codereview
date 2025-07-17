"""

~~~

{
  "size": 0,
  "query": {
    "range": {
      "load_timestamp": {
        "gte": "now-1h/h",    // Adjust your time range as needed
        "lte": "now"
      }
    }
  },
  "aggs": {
    "docs_per_minute": {
      "date_histogram": {
        "field": "load_timestamp",
        "fixed_interval": "1m",
        "min_doc_count": 0    // include empty buckets if you want
      }
    },
    "avg_docs_per_minute": {
      "avg_bucket": {
        "buckets_path": "docs_per_minute._count"
      }
    }
  }
}




~~~

"""
