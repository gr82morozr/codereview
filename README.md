
"""




PUT _ingest/pipeline/validate_nested_fields
{
  "processors": [
    {
      "script": {
        "description": "Loop through predefined repeated fields and clean subfields with invalid values",
        "lang": "painless",
        "source": """
          def targetFields = [
            ['path': 'qualifications', 'fields': ['year', 'level', 'status']],
            ['path': 'experiences',     'fields': ['from', 'to', 'position']]
          ];
          
          for (def config : targetFields) {
            def path = config.path;
            def fields = config.fields;
            if (ctx.containsKey(path) && ctx.get(path) instanceof List) {
              for (def item : ctx.get(path)) {
                for (def field : fields) {
                  if (item.containsKey(field)) {
                    def val = item.get(field);
                    // Example validation: remove if null or empty string
                    if (val == null || (val instanceof String && val.trim().length() == 0)) {
                      item.remove(field);
                    }
                  }
                }
              }
            }
          }
        """
      }
    }
  ]
}





"""
