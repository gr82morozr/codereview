"""

PUT _ingest/pipeline/date_and_bool_cleaner
{
  "processors": [
    {
      "script": {
        "description": "Remove empty/null/missing date fields",
        "lang": "painless",
        "source": """
          String[] dateFields = ['created_at', 'updated_at', 'deleted_at'];
          for (String f : dateFields) {
            if (!ctx.containsKey(f) || ctx[f] == null || ctx[f] == '') {
              ctx.remove(f);
            }
          }
        """
      }
    },
    {
      "script": {
        "description": "Convert Y/N to boolean",
        "lang": "painless",
        "source": """
          String[] ynFields = ['is_active', 'is_deleted'];
          for (String f : ynFields) {
            if (ctx.containsKey(f) && ctx[f] != null) {
              String val = ctx[f].toString().toUpperCase();
              if (val == 'Y') {
                ctx[f] = true;
              } else if (val == 'N') {
                ctx[f] = false;
              }
            }
          }
        """
      }
    }
  ]
}




"""
