"""



 {
  "script": {
    "lang": "painless",
    "source": """
      def dateFields = [
        ['Date of Birth'],
        ['Deceased Date'],
        ['Created'],
        ['Updated'],
        ['FaCS NQSC Worker Qualification', 'Created'],
        ['FaCS NQSC Worker Qualification', 'Updated'],
        ['FaCS NQSC Worker Identification', 'Updated']
        // Add more as needed
      ];

      for (def path : dateFields) {
        def obj = ctx;
        for (int i = 0; i < path.length - 1; i++) {
          if (obj.containsKey(path[i]) && obj[path[i]] instanceof Map) {
            obj = obj[path[i]];
          } else {
            obj = null;
            break;
          }
        }
        if (obj != null) {
          def key = path[path.length - 1];
          if (!obj.containsKey(key) || obj[key] == null || obj[key] == '') {
            obj.remove(key);
          }
        }
      }
    """
  }
}

      

"""
