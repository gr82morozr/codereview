"""

~~~


We’d like to request a 2-week (10 working days) test window in the SPT environment to conduct testing related to syncing Worker data to Elasticsearch indices. Given SPT’s production-like data volume, it is the most suitable environment to validate the timing and sizing required for the December release.

Please note the following:

This activity will not involve any Siebel data changes.

We will be syncing approximately 1.3 million Worker records to new Elasticsearch indices.

The code migration is independent of the September release code and will not impact existing components.

The changes include a new Workflow, new EAI Dispatch Rule, and new Runtime Event.

Tentative test plan:

Days 1–3
• Set up Elasticsearch indices
• Deploy Siebel changes
• Perform initial sync and validate basic data flow

Days 4–10
• Test real-time and bulk data synchronization in several cycles
• Assess performance and timing under expected data volumes
• Fine-tune Siebel parameters and code if necessary
• Perform final validation and cleanup

Could you please confirm a suitable window for us to proceed?

Appreciate your support!



~~~

"""
