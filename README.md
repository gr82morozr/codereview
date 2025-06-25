"""


def nameField = 'ssss';

if (ctx.containsKey(nameField)) {
    def val = ctx[nameField];

    // Handle array of objects
    if (val instanceof List) {
        for (int i = 0; i < val.size(); i++) {
            def item = val[i];
            def fn = item.containsKey('First Name') && item['First Name'] != null ? item['First Name'] : '';
            def mn = item.containsKey('Middle Name') && item['Middle Name'] != null ? item['Middle Name'] : '';
            def ln = item.containsKey('Last Name') && item['Last Name'] != null ? item['Last Name'] : '';
            def parts = [];
            if (fn.length() > 0) parts.add(fn);
            if (mn.length() > 0) parts.add(mn);
            if (ln.length() > 0) parts.add(ln);
            item['Full Name'] = String.join(' ', parts);
        }
    }

    // Handle single object
    else if (val instanceof Map) {
        def item = val;
        def fn = item.containsKey('First Name') && item['First Name'] != null ? item['First Name'] : '';
        def mn = item.containsKey('Middle Name') && item['Middle Name'] != null ? item['Middle Name'] : '';
        def ln = item.containsKey('Last Name') && item['Last Name'] != null ? item['Last Name'] : '';
        def parts = [];
        if (fn.length() > 0) parts.add(fn);
        if (mn.length() > 0) parts.add(mn);
        if (ln.length() > 0) parts.add(ln);
        item['Full Name'] = String.join(' ', parts);
    }
}

"""
