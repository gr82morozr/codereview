"""


The Cycle #1 relevance tuning is now complete and has been deployed to BPA. Key changes in this run include:

Ngram updated to 4 – to reduce excessive partial matches.
→ Analyzer ngram size adjusted.

Siebel ROW_ID is now treated as a whole term – e.g., "4-5JSRQVK" won't be split.
→ Added a char_filter to replace "_" with "-" to prevent tokenization.

Wildcard support for ROW_ID – e.g., you can now search like "4-5JI*".
→ Wildcard search added in the search template for relevant ID fields.

Date fields are now searchable – e.g., "12/13/2019" in mm/dd/yyyy format.
→ Date fields are also indexed as text to support this.

Work full name is now searchable –
→ Full name populated via the ingest pipeline.

Additionally, the highlight match issue has been fixed (thanks to Jarl!).

Please test this version and let me know if you run into any issues.


"""
