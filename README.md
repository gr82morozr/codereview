"""

~~~


A couple of key points regarding search functionality:

Search by expression is not currently implemented.
For example:

FieldName = Value

"john wick" AND "john yossarian" NOT "sam john"

Currently, the search term is passed to Elasticsearch as a plain string.

To support expression-based search, the following are required:

The frontend must explicitly allow users to indicate they are entering an expression—not just a simple search term.

The expression syntax must conform to Elasticsearch’s query string syntax.

Let me know if you need further clarification.


~~~

"""
