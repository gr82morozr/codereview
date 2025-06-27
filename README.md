"""


tabs = ["Overview", "Details", "Logs", "Settings"]  # This can be dynamic

html = """
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Dynamic Tabs</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 0; }
        .tab-container { display: flex; border-bottom: 1px solid #ccc; background: #f1f1f1; }
        .tab { padding: 12px 20px; cursor: pointer; border: none; background: none; outline: none; }
        .tab.active { background: white; border-bottom: 2px solid #007BFF; font-weight: bold; }
        .tab-content { display: none; padding: 20px; }
        .tab-content.active { display: block; }
    </style>
</head>
<body>

<div class="tab-container">
"""

# Generate tabs
for i, name in enumerate(tabs):
    active = "active" if i == 0 else ""
    html += f'<button class="tab {active}" onclick="openTab(event, \'tab{i}\')">{name}</button>\n'

html += "</div>\n"

# Generate tab content
for i, name in enumerate(tabs):
    active = "active" if i == 0 else ""
    html += f'<div id="tab{i}" class="tab-content {active}">\n'
    html += f'  <h2>{name} Content</h2>\n'
    html += f'  <p>This is the content of the "{name}" tab.</p>\n'
    html += '</div>\n'

# Add JS for tab switching
html += """
<script>
function openTab(evt, tabId) {
    var i, content, tabs;
    content = document.getElementsByClassName("tab-content");
    tabs = document.getElementsByClassName("tab");

    for (i = 0; i < content.length; i++) {
        content[i].classList.remove("active");
    }
    for (i = 0; i < tabs.length; i++) {
        tabs[i].classList.remove("active");
    }

    document.getElementById(tabId).classList.add("active");
    evt.currentTarget.classList.add("active");
}
</script>
</body>
</html>
"""

# Save or print the result
with open("tabs.html", "w", encoding="utf-8") as f:
    f.write(html)



""""
