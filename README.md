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
        ],
        [
          'path': 'order.items',
          'old_names': ['sku_old'],
          'new_names': ['sku']
        ]
      ];

      for (def rule : rename_fields) {
        def path = rule.path.split('\\.');
        def obj = ctx;
        // Traverse to the parent object
        for (int i = 0; i < path.length - 1; i++) {
          if (obj.containsKey(path[i]) && obj[path[i]] != null) {
            obj = obj[path[i]];
          } else {
            obj = null;
            break;
          }
        }
        if (obj != null && obj.containsKey(path[-1]) && obj[path[-1]] != null) {
          def target = obj[path[-1]];
          // If it's a List, rename in each item
          if (target instanceof List) {
            for (item in target) {
              for (int j = 0; j < rule.old_names.length; j++) {
                def old_name = rule.old_names[j];
                def new_name = rule.new_names[j];
                if (item.containsKey(old_name) && item[old_name] != null) {
                  item[new_name] = item.remove(old_name);
                }
              }
            }
          // If it's a Map/Object, rename directly
          } else if (target instanceof Map) {
            for (int j = 0; j < rule.old_names.length; j++) {
              def old_name = rule.old_names[j];
              def new_name = rule.new_names[j];
              if (target.containsKey(old_name) && target[old_name] != null) {
                target[new_name] = target.remove(old_name);
              }
            }
          }
          // If not a List or Map, do nothing (skip)
        }
      }
    """
  }
}



~~~

"""
