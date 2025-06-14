
"""





          def validations = [
            ['path': '/',              'subfields': ['source_system', 'load_timestamp', 'user_status']],
            ['path': 'qualifications', 'subfields': ['year', 'level', 'status']],
            ['path': 'experiences',    'subfields': ['from', 'to', 'position']]
          ];

          def cleanField = (Map target, List fields) -> {
            for (def field : fields) {
              if (target.containsKey(field)) {
                def val = target.get(field);
                if (val == null || (val instanceof String && val.trim().length() == 0)) {
                  target.remove(field);
                }
              }
            }
          };

          for (def config : validations) {
            def path = config.path;
            def subfields = config.subfields;

            if (path == '/') {
              cleanField(ctx, subfields);
              continue;
            }

            if (!ctx.containsKey(path)) continue;
            def value = ctx.get(path);
            def items = value instanceof List ? value : [value];

            for (def item : items) {
              if (item instanceof Map) {
                cleanField(item, subfields);
              }
            }

            if (!(value instanceof List)) {
              ctx.put(path, items[0]);
            }
          }
     





"""
