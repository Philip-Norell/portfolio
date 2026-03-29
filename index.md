---
layout: default
title: Philip Norell's Portfolio
---

# Philip Norell: GIS Portfolio

<style>
/* Container Adjustments */
#main_content_wrap.outer {
  max-width: 4000px;
  width: 100%;
  margin-left: auto;
  margin-right: auto;
}

#main_content.inner {
  max-width: 1000px;
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
  font-family: inherit;
  font-size: 1rem;
}

.tab-button.active {
  border-bottom: 3px solid #4CAF50;
  color: #4CAF50;
}

.tab-content {
  display: none;
  padding-top: 10px;
}

.tab-content.active {
  display: block;
}

/* Map specific styling */
.map-frame {
  border: 1px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  background: #fff;
}
</style>

<div class="tabs">
  <button class="tab-button active" onclick="openTab(event, 'tab1')">About Me</button>
  <button class="tab-button" onclick="openTab(event, 'tab2')">ArcGIS API</button>
  <button class="tab-button" onclick="openTab(event, 'tab3')">Python & Visualization</button>
  <button class="tab-button" onclick="openTab(event, 'tab4')">JSON/Data Structures</button>
</div>

<div id="tab1" class="tab-content active">
  <p>Welcome to my portfolio! I'm a GIS technician specializing in Enterprise and Web GIS. I enjoy solving difficult problems with a bit of research, a bit of code, and a bit of elbow grease!</p>
  <p>I'm passionate about urban design, and I'm looking to work and live in a forward-thinking city I can be proud of as a resident and an employee.</p>
</div>

<div id="tab2" class="tab-content">
  {% capture notebook %}
    {% include PNorell_Dependency_Automator.md %}
  {% endcapture %}
  {{ notebook | markdownify }}
</div>

<div id="tab3" class="tab-content">
  <div style="margin-bottom: 15px;">
    <h3>National Zoning Restrictiveness Index</h3>
    <p>This interactive map visualizes land-use regulations across 4,000+ municipalities using data retrieved via the ArcGIS REST API. 
       <strong>Click on any marker</strong> to view the specific index score for that city.</p>
  </div>

  <div class="map-frame">
    <iframe 
      src="zri_index_map.html" 
      width="100%" 
      height="750px" 
      style="border:none; display:block;" 
      loading="lazy">
    </iframe>
  </div>
  <p style="margin-top: 10px;"><small><em>Note: High values indicate more restrictive zoning environments.</em></small></p>
</div>

<div id="tab4" class="tab-content">
<pre><code>
{
  "focus": "Enterprise GIS",
  "skills": ["Python", "ArcPy", "SQL", "Leaflet"],
  "goal": "Urban Policy Optimization"
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
