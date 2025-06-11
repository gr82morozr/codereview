"""

Thanks again for the helpful suggestions — really appreciate your input.

This index is purely for staging, holding raw data from Siebel as the source of truth for downstream processing. All fields are mapped as text to keep things flexible and tolerant of varying data types, since it's just for storage.

A separate, search-optimized index will be created from this. In that one, fields like IDs will be mapped as keyword, and dates will use proper date types as you recommended.

Since the data isn’t time-series, we haven’t applied ILM or data streams. I did include a load_timestamp field though, mainly to support delta reindexing and to help track sync latency for the real-time interface.

Thanks again for your review!



"""
