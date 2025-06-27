"""



<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Multi-Tab Layout</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 0; }
        .tab-container {
            display: flex;
            border-bottom: 1px solid #ccc;
            background-color: #f1f1f1;
        }
        .tab {
            padding: 12px 20px;
            cursor: pointer;
            border: none;
            background: none;
            outline: none;
        }
        .tab.active {
            background: white;
            border-bottom: 2px solid #007BFF;
            font-weight: bold;
        }
        .tab-content {
            display: none;
            padding: 20px;
        }
        .tab-content.active {
            display: block;
        }
    </style>
</head>
<body>

<div class="tab-container">
    <button class="tab active" onclick="openTab(event, 'tab0')">Overview</button>
    <button class="tab" onclick="openTab(event, 'tab1')">Details</button>
    <button class="tab" onclick="openTab(event, 'tab2')">Logs</button>
</div>

<div id="tab0" class="tab-content active">
    <div id="content_0">Loading Overview content...</div>
</div>
<div id="tab1" class="tab-content">
    <div id="content_1">Loading Details content...</div>
</div>
<div id="tab2" class="tab-content">
    <div id="content_2">Loading Logs content...</div>
</div>

<script>
function openTab(evt, tabId) {
    const contents = document.getElementsByClassName("tab-content");
    const tabs = document.getElementsByClassName("tab");

    for (let i = 0; i < contents.length; i++) {
        contents[i].classList.remove("active");
    }
    for (let i = 0; i < tabs.length; i++) {
        tabs[i].classList.remove("active");
    }

    document.getElementById(tabId).classList.add("active");
    evt.currentTarget.classList.add("active");
}
</script>

</body>
</html>




"""
