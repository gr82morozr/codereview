"""

~~~


We’d like to request a 4-week test window in the SPT environment to conduct testing related to syncing Worker data to Elasticsearch indices. Given SPT’s production-like data volume, it is the most suitable environment to validate the timing and sizing required for the December release.

Please note the following:

This activity will not involve any Siebel data changes.

We will be syncing approximately 1.3 million Worker records to new Elasticsearch indices.

The code migration is independent of the September release code and will not impact existing components.

The changes include a new Workflow, new EAI Dispatch Rule, and new Runtime Event.

Tentative 4-week test plan:

Week 1 – Set up Elasticsearch indices, deploy Siebel changes, and perform initial sync

Weeks 2–3 – Test real-time and bulk data synchronization in multiple cycles; assess performance/timing under varying data loads; fine-tune code and Siebel component parameters if needed

Week 4 – Final validation, cleanup, and rollback of Siebel changes (if required)

Could you please confirm a suitable window for us to proceed?



~~~

"""
