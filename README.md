"""



  def nestedDateFields = [
        ['FaCS NQSC Worker Qualification', 'Updated'],
        ['FaCS NQSC Worker Qualification', 'Created'],
        ['FaCS NQSC Worker Qualification', 'Date Completed'],
        ['FaCS NQSC Worker Identification', 'Created'],
        ['FaCS NQSC Worker Identification', 'Updated']
        // Add more as needed
      ];

      for (def pair : nestedDateFields) {
        def parent = pair[0];
        def child = pair[1];
        if (ctx.containsKey(parent)) {
          def sub = ctx[parent];
          if (sub instanceof Map && (!sub.containsKey(child) || sub[child] == null || sub[child] == '')) {
            sub.remove(child);
          }
        }
      }
      

"""
