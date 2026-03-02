---
layout: default
title: Home
---

# Welcome to My Website

<style>
.tabs {
  display: flex;
  border-bottom: 2px solid #ddd;
  margin-bottom: 1rem;
}

.tab-button {
  padding: 10px 20px;
  cursor: pointer;
  border: none;
  background: none;
  font-weight: bold;
}

.tab-button.active {
  border-bottom: 3px solid #4CAF50;
  color: #4CAF50;
}

.tab-content {
  display: none;
}

.tab-content.active {
  display: block;
}
</style>

<div class="tabs">
  <button class="tab-button active" onclick="openTab('tab1')">Notebook</button>
  <button class="tab-button" onclick="openTab('tab2')">Python</button>
  <button class="tab-button" onclick="openTab('tab3')">JSON</button>
</div>

<div id="tab1" class="tab-content active">
  {% capture notebook %}
  {% include AGOL_Dependency_Automator_GitHub.md %}
  {% endcapture %}
  {{ notebook | markdownify }}
</div>

<div id="tab2" class="tab-content">
<pre><code>
# Example Python Code
print("Hello World")
</code></pre>
</div>

<div id="tab3" class="tab-content">
<pre><code>
{
  "example": true
}
</code></pre>
</div>

<script>
function openTab(tabId) {
  var contents = document.querySelectorAll('.tab-content');
  var buttons = document.querySelectorAll('.tab-button');

  contents.forEach(c => c.classList.remove('active'));
  buttons.forEach(b => b.classList.remove('active'));

  document.getElementById(tabId).classList.add('active');
  event.target.classList.add('active');
}
</script>
