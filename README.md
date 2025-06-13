"""
Status Update
Data Load Index:

The data load index is completed.

The real-time sync interface between the NQSC worker and Elasticsearch is working (WF change not yet delivered to int_future workspace, so RTE not set up to invoke the workflow).

Noticed: real-time sync latency is currently ≥12s.

Search Index & Ingest Pipeline:

Work on search index mapping and the ingest pipeline is in progress.

Query Logic (DSL with Search Template):

Development of the query logic using a search template is ongoing and will be aligned with the finalized search index mapping.

To Do
Update Search UI to display results from the new query.

Investigate and reduce real-time sync latency (per Designer feedback).

Support for bulk data loading.

Suggestions
Use search templates to encapsulate complex query logic in Elasticsearch. This allows the frontend (Search UI) to simply pass required parameters and invoke the template. Future changes to query logic/tuning can be managed within Elasticsearch, with no need for frontend code changes.



"""
