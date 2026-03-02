---
layout: default
title: Home
---

# Welcome to My Website

<style>
/* Target the main wrapper in Minimal theme */
.wrapper {
  max-width: 1400px !important;  /* widen container */
  width: 95% !important;          /* responsive width */
  margin-left: auto !important;   /* center horizontally */
  margin-right: auto !important;
}
</style>

<div class="tabs">
  <button class="tab-button active" onclick="openTab(event, 'tab1')">Notebook</button>
  <button class="tab-button" onclick="openTab(event, 'tab2')">Python</button>
  <button class="tab-button" onclick="openTab(event, 'tab3')">JSON</button>
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
function openTab(evt, tabId) {
  var contents = document.querySelectorAll('.tab-content');
  var buttons = document.querySelectorAll('.tab-button');

  contents.forEach(c => c.classList.remove('active'));
  buttons.forEach(b => b.classList.remove('active'));

  document.getElementById(tabId).classList.add('active');
  evt.currentTarget.classList.add('active');
}
</script>
