"""


{
  "script": {
    "description": "Remove empty date fields or reformat MM/dd/yyyy to dd/MM/yyyy (preserve time)",
    "lang": "painless",
    "source": """
      def targetFields = [
        [ "path" : "",           "fields" : ["datefieldA", "datefieldB"] ],
        [ "path" : "ChildData1", "fields" : ["datefieldC", "datefieldD"] ],
        [ "path" : "ChildData2", "fields" : ["datefieldF", "datefieldE"] ]
      ];

      for (entry in targetFields) {
        def path = entry['path'];
        def fields = entry['fields'];

        def container = path == "" ? ctx : (ctx.containsKey(path) ? ctx[path] : null);
        if (container == null) continue;

        for (field in fields) {
          if (!container.containsKey(field) || container[field] == null || container[field].toString().trim() == "") {
            container.remove(field);
            continue;
          }

          def value = container[field].toString();
          def m = /(?<mm>\d{1,2})\/(?<dd>\d{1,2})\/(?<yyyy>\d{4})(?<time>[\sT].+)?/.matcher(value);

          if (m.matches()) {
            def newDate = m.group("dd") + "/" + m.group("mm") + "/" + m.group("yyyy");
            if (m.group("time") != null) {
              newDate += m.group("time");
            }
            container[field] = newDate;
            // Optionally also store as text version
            // container[field + ".as_text"] = newDate;
          }
        }
      }
    """
  }
}



"""
