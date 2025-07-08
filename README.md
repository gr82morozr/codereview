"""

~~~


{
  "script": {
    "lang": "painless",
    "source": """
      def rename_fields = [
        [
          'path': 'customer.contacts',
          'old_names': ['old_name1', 'old_name2'],
          'new_names': ['new_name1', 'new_name2']
        ]
        // You can add more entries here if needed
      ];
      
      for (def rule : rename_fields) {
        def path = rule.path.split('\\.');
        def obj = ctx;
        // Traverse to the array's parent
        for (int i = 0; i < path.length - 1; i++) {
          if (obj.containsKey(path[i])) {
            obj = obj[path[i]];
          } else {
            obj = null;
            break;
          }
        }
        if (obj != null && obj.containsKey(path[-1]) && obj[path[-1]] != null) {
          def arr = obj[path[-1]];
          // For every object in the array, perform renames
          for (item in arr) {
            for (int j = 0; j < rule.old_names.length; j++) {
              def old_name = rule.old_names[j];
              def new_name = rule.new_names[j];
              if (item.containsKey(old_name)) {
                item[new_name] = item.remove(old_name);
              }
            }
          }
        }
      }
    """
  }
}


~~~

"""
