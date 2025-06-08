"""
POST _reindex
{
  "source": {
    "index": "indexA"
  },
  "dest": {
    "index": "indexB"
  },
  "script": {
    "lang": "painless",
    "source": """
      Map newDoc = new HashMap();
      newDoc['field1'] = ctx._source['field1'];
      newDoc['field2'] = ctx._source['field2'];
      newDoc['field3'] = ctx._source['field3'];
      ctx._source = newDoc;
    """
  }
}

"""
