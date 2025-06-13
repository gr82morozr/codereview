"""




      for (String f : dateFields) {
        def parts = f.splitOnToken('\\.');
        def obj = ctx;
        for (int i = 0; i < parts.length - 1; i++) {
          if (obj.containsKey(parts[i])) {
            obj = obj[parts[i]];
          } else {
            obj = null;
            break;
          }
        }
        if (obj != null) {
          def lastKey = parts[parts.length - 1];
          if (!obj.containsKey(lastKey) || obj[lastKey] == null || obj[lastKey] == '') {
            obj.remove(lastKey);
          }
        }
      }
      

"""
