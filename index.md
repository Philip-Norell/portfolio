---
layout: default
title: Philip Norell's Portfolio
---

# Contents

<style>
#main_content_wrap.outer {
  max-width: 4001px;    /* widen outer container */
  width: 100%;
  margin-left: auto;    /* center horizontally */
  margin-right: auto;
}

#main_content.inner {
  max-width: 1000px;    /* widen inner container */
  width: 95%;
  margin-left: auto;
  margin-right: auto;
}

/* Tabs styling */
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
  <button class="tab-button active" onclick="openTab(event, 'tab1')">About Me</button>
  <button class="tab-button" onclick="openTab(event, 'tab2')">ArcGIS API</button>
  <button class="tab-button" onclick="openTab(event, 'tab3')">Python</button>
  <button class="tab-button" onclick="openTab(event, 'tab4')">JSON</button>
</div>

<div id = "tab1" class = "tab-content active">
<p>Welcome to my portfolio! I'm a GIS technician specializing in Enterprise and Web GIS. I enjoy solving difficult problems with a bit of research, a bit of code, and a bit of elbow grease!</p>
<p>I'm passionate about urban design, and I'm looking to work and live in a forward-thinking city I can be proud of as a resident and an employee.</p>
  
</div>

<div id = "tab2" class = "tab-content">
  {% capture notebook %}
  {% include PNorell_Dependency_Automator.md %}
  {% endcapture %}
  {{ notebook | markdownify }}
</div>

<div id="tab3" class="tab-content">
<pre><code>
{
  "example": true
}
</code></pre>
</div>

<div id="tab4" class="tab-content">
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
