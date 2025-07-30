"""

~~~



Before I go on leave, here’s a summary of the current status and key technical points for the Elasticsearch project.

[SPT Testing] – 1M Records Data Load (Total ~1.3M Records)
The full data load test with 1 million records was successfully completed over the weekend (~2 days).

Initial throughput was around 1000 docs/min, but degraded significantly over time to <100 docs/min.

Analysis indicates the bottleneck is the SPT Elasticsearch cluster, which only has 2 nodes (compared to 3 nodes in BPA).

Production, with 3 Elasticsearch nodes, is expected to perform better and more consistently.

Issues Identified During Testing
Around 300k Worker records were not loaded into Elasticsearch.

Approximately 0.02% of documents contain illegal UTF-8 characters, which caused the REST API call to fail, and these documents were not indexed.

Elasticsearch Throughput Degradation

Likely caused by resource pressure, particularly I/O bottlenecks, during sustained bulk indexing.

Requires deeper investigation and isolated benchmarking.

[Relevance Tuning in BPA]
Advised the Design team to pause further tuning in BPA.

Relevance tuning will continue in higher environments with more realistic datasets.

[Technical Recommendation for December Release]
Exception Handling in EAI Framework

Add robust exception handling in the current EAI framework, particularly during multi-index updates (both dataload and search indices), to avoid silent failures and improve traceability.

[To-Do List / Next Steps]
Overnight Load Test (Wed or Thurs, pending approval):

Run smaller batch jobs to capture additional exception scenarios during Siebel-to-ES sync.

Key goal: identify root cause of the 300k missing records from the weekend load.

UTF-8 Character Handling:

Investigate and implement a workaround or sanitation logic to handle non-English UTF-8 characters that currently break the data load.

Elasticsearch Throughput Benchmarking:

Conduct isolated ingestion tests (outside of Siebel) to assess whether Elasticsearch can maintain the expected throughput under load.

Great thanks to  for his strong support with the SPT testing and detailed investigation during the run – his help has been much appreciated.

 – please feel free to add more details if anything is missed and keep Andrew and Aravind updated on the ongoing investigations.


~~~

"""
