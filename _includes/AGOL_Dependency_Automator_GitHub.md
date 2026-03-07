AGOL Dependency Automator

**Summary of Purpose**  

This script mostly fulfills the same role as Qonda reports. It's a generalized data governance solution that utilized the ArcGIS API for Python to create data lineage documentation. 
It requires only one manually created input that maps feature classes to REST/referenced services on ArcServers. It was created before I or anyone else in our shop was aware of Qonda,
and provides some benefits over Qonda such as creating data lineage snapshots. 

**Notes on Script**

Method
        There are two avenues of data capture in this script.    
    First, it compiles a dictionary of all webmaps in AGOL and parses their related JSONs to extract the services they depend upon. 
    Those services are then matched to a dictionary of feature services and feature classes, producing a dictionary structured like
    {webmap : {service : [feature classes]}}. This is then transformed into a pandas dataframe, transposed, and then written as a 
    formatted Excel sheet. Second, the same process is performed on a dictionary of experiences, creating a dictionary strucutred as 
    {experience : {webmap : {service : [feature classes]}}.

Inputs:
        This script takes inputs from two sources. The first is directly from Portal/AGOL via the ArcGIS API, and the second is a 
    manually created document detailing which feature classes services consume. The second document must be created and maintained 
    manually, but such maintenance will only be required infrequently due to how seldom feature classes (as well as referenced services)
    are added or removed from SDE. The creation process for the feature class input sheet is detailed in the documentation. 

Limitations: 
        This script does not capture which services experiences consume directly without a webmap intermediary.
    To my knowledge, this is possible but may be difficult. Therefore, since our standard practice is for experiences to only
    consume webmaps, I didn't deem this necessary. Furthermore, the orphan services sheet produced does not account for services
    drawn upon by solutions. 

Misc:
        The dictionary-building code chunks will often return error text along the lines of "Service (ID String) does not exist."
    These are deleted services that retain their itemID but have all attributes such as title and URL erased. If the ID is plugged 
    into a standard AGOL/Portal item URL, it will say the service is deleted or unavailable. 

Documentation location: "filepath\Creating Input Data for Dependency Visualizer.docx"

        

    
        


```python
from arcgis.gis import GIS
import pandas as pd
import re
import os
import datetime
import xlsxwriter as xl
import numpy as np

date = datetime.date.today()
formatted_date_mdy = date.strftime("%m_%d_%Y")
os.mkdir(fr"filepath\AGOL_Dependencies_{formatted_date_mdy}")

```


```python

# Imports experiences, apps and maps from AGOL and converts them into dictionaries. 
# Apps are not used in the code, but it could be adapted such that apps replace experiences.
# Ai-assisted.

ExpTitle = []
ExpID = []
MapTitle = []
MapID = []
servtitle = [] 
servid = []
servtype = []
servurl = []
servowner = []
servcreated = []
servmod = []
servcreated_unix = []
servmod_unix = []
dashtitle = []
dashid = []

portal_url = 
username = 
password =    

try:
    gis = GIS(portal_url, username, password)
    print(f"Successfully connected to {gis.properties.portalName}")
except Exception as e:
    print(f"Error connecting to the portal: {e}")
    exit()

web_exps = gis.content.search(query = "type:Web Experience", item_type = "Web Experience", max_items = 1000)
web_maps = gis.content.search(query = "type:Web Map", item_type = "Web Map", max_items = 1000)
feature_services = gis.content.search(query = "type:Feature Service", item_type = "Feature Service", max_items = 5000)
map_services = gis.content.search(query = "type:Map Service", item_type = "Map Service", max_items = 5000)
vector_services = gis.content.search(query = "type:Vector Tile Service", item_type = "Vector Tile Service", max_items = 5000)
dashboards = gis.content.search(query = "type:Dashboard", item_type = "Dashboard", max_items = 1000)


if web_exps:
    for web_exp in web_exps:
        ExpTitle.append(web_exp.title)
        ExpID.append(web_exp.id)
else:
    print("No experiences found in the portal.")

if web_maps:
    for web_map in web_maps:
        MapTitle.append(web_map.title)
        MapID.append(web_map.id)   
            
else:
    print("No Web Maps found in the portal.") 

if feature_services:
    for service in feature_services:
        servtitle.append(service.title)
        servid.append(service.id)
        servtype.append(service.type)
        servurl.append(service.url)
        servowner.append(service.owner)
        servcreated_unix.append(service.created)
        servmod_unix.append(service.modified)
        
if map_services:
    for service in map_services:
        servtitle.append(service.title)
        servid.append(service.id)
        servtype.append(service.type)
        servurl.append(service.url)
        servowner.append(service.owner)
        servcreated_unix.append(service.created)
        servmod_unix.append(service.modified)
        
if vector_services:
    for service in vector_services:
        servtitle.append(service.title)
        servid.append(service.id)
        servtype.append(service.type)
        servurl.append(service.url)
        servowner.append(service.owner)
        servcreated_unix.append(service.created)
        servmod_unix.append(service.modified)

if dashboards:
    for dashboard in dashboards:
        dashtitle.append(dashboard.title)
        dashid.append(dashboard.id)

        
expdict = dict(zip(ExpTitle, ExpID)) 
mapdict = dict(zip(MapTitle, MapID)) 
dashdict = dict(zip(dashtitle, dashid))
```


```python
for each in servcreated_unix: # Converts unix millisecond timestamp to date
    date_s = each / 1000
    date = datetime.date.fromtimestamp(date_s)
    servcreated.append(date)

for each in servmod_unix:
    date_s = each / 1000
    date = datetime.date.fromtimestamp(date_s)
    servmod.append(date)
    

services_list = [] 
for name, ID, typ, url, owner, created, modified in zip(servtitle, servid, servtype, servurl, servowner, servcreated, servmod):
    string = name + " | " + owner + " | " + f'{created}' + " | " + f'{modified}' + " | " + ID + " | " + typ + " | " + url
    services_list.append(string)
```


```python
dependency_df = pd.read_csv(r"filepath\FeatureServiceFeatureClassDependencies.csv")

temp_df = dependency_df.T.values
temp_list = []
for each in temp_df:
    x = []
    for val in each:
        if val != "N":
            x.append(val)      
    
    temp_list.append(x)

keylist = []
vallist = []
for each in temp_list:
    key = each[0]
    keylist.append(key)
    val = each[1:]
    vallist.append(val)

fc_dict = dict(zip(keylist, vallist))
```


```python
# Defining Functions

# Creates a dictionary of experiences/dashboards and webmaps. Used on the JSONs of dashboards and experiences.
# Defining Functions

# Creates a dictionary of experiences/dashboards and webmaps. Used on the JSONs of dashboards and experiences.
def find_web_maps(data,found_values=None): 
    if found_values is None:
        found_values = []

    if isinstance(data, dict):
        for key, value in data.items():
            if data.get('type') == 'WEB_MAP':
                if 'itemId' in data:
                    dep_item = gis.content.get(data['itemId'])
                    if dep_item:
                        info_str = (f"{dep_item.title}, Item ID: {dep_item.id}") 
                        found_values.append(info_str)
            find_web_maps(value,found_values)

    
    return found_values

def dash_find_web_maps(data,found_values=None): 
    if found_values is None:
        found_values = []

    if isinstance(data, dict):
        if "itemId" in data:
            
            dep_id = data.get('itemId')            
            if dep_id is not None:
                dep_item = gis.content.get(dep_id)
                if dep_item:
                    info_str = info_str = (f"{dep_item.title}, Item ID: {dep_item.id}, {dep_item.type}")
                    found_values.append(info_str)
                else:
                    print(f'Service {dep_id} does not exist')
                    
        for key, value in data.items():
            dash_find_web_maps(value, found_values)
            
    elif isinstance(data, list): 
        for each in data:
            dash_find_web_maps(each, found_values)
    
    return found_values



# Recursively searches though map dependency JSON for item IDs and rest service URLs and creates a list of dependencies.
# Used within a for loop of web maps.

def dependency_recursion(data,found_values=None):
    if found_values is None:
        found_values = []

    if isinstance(data, dict):
        if 'itemId' in data:
            dep_id = data.get('itemId')
            dep_item = gis.content.get(dep_id)
            if dep_item:
                info_str = (f"{dep_item.title} || {dep_item.id} || {dep_item.type} || {dep_item.url}")
                found_values.append(info_str)
            else:
                print(f'Service {dep_id} does not exist')
        elif 'url' in data:
            info_str = data.get('url')
            if r'/image' not in info_str.lower() and r'/mapviewer' not in info_str.lower() and not re.fullmatch(r'[\w\W]{32,38}', info_str):
                found_values.append(info_str)
            
        for key, value in data.items():
            dependency_recursion(value, found_values)
            
    elif isinstance(data, list): 
        for each in data:
            dependency_recursion(each, found_values)
    
    return found_values
    
# AI-created alphabet generator function
def make_alphabet(n):
    labels = []
    while len(labels) < n:
        num = len(labels)
        label = ""
        while True:
            num, rem = divmod(num, 26)
            label = chr(65 + rem) + label
            if num == 0:
                break
            num -= 1
        labels.append(label)
    return labels

# Xlsxwriter formatting function
def create_new_sheet(writer_object, dataframe, sheet_title, index_str):
    dataframe = dataframe.fillna("")
    dataframe.to_excel(writer, sheet_name = sheet_title)
    sheet_id = writer.sheets[sheet_title]
    
    headers_list = []
    for each in dataframe.columns:
        headers = {'header' : each}
        headers_list.append(headers)    
    headers_list[0:0] = [{'header' : index_str}]

    alphabet = make_alphabet(len(headers_list))
    rowcount = dataframe.shape[0] + 1

    sheet_id.add_table(f'A1:{alphabet[-1]}{rowcount}', {'columns' : headers_list})

    return sheet_id
```


```python
# Converts mapdict's keys into a better format for later use
templist = []
mapdict2 = {}
for each in mapdict:
    deptemp = gis.content.get(mapdict[each])
    infostrtemp = (f"{deptemp.title}, Item ID: {deptemp.id}")
    templist.append(infostrtemp)
    mapdict2[infostrtemp] = mapdict[each]
```


```python
map_dependency_dict = {}
for key in mapdict2:
    innermost_dict = {}
    try:
        mapItem = gis.content.get(mapdict2[key]) #finds webmap associated with the id value in mapdict.
        dependencies = mapItem.get_data(try_json = True)['operationalLayers'] #returns a dictionary with details on supporting services. try_json is optional but may help convert the data to a dictionary.
        dep_info_list = []
        
        dep_info_list = dependency_recursion(dependencies) #Pulls supporting services out of dependency JSON and adds to list
                    
        for each in dep_info_list:
            match_found = False
            for x, y in fc_dict.items():
                if x in each:
                    innermost_dict[each] = y
                    match_found = True
                    break
                if not match_found:
                    innermost_dict[each] = ["This Service is Hosted on AGOL or is a Raster Image"]
        
        map_dependency_dict[key] = innermost_dict #writes keys and values to empty dictionary.
    except Exception as e:
        print(f'{key} {e}')
```


```python
experience_dependency_dict = {}

for key in expdict:
    try:
        experience = gis.content.get(expdict[key])
        func_in = experience.get_data()
        final = find_web_maps(func_in)
        experience_dependency_dict[key] = final
    except Exception as e:
        print(f'{key}{e}')

```


```python
# Creates a nested dictionary detailing all dependency levels.
# The dictionary is formatted as follows: {Experience: {Web Map: {Service: [Feature Class]}}}


exp_dependency_dict = {}
for key in experience_dependency_dict: #Looping over experiences
    inner_dict = {}

    
    #Finds the dependencies of each webmap associated with each experience    
    try:
        for webmapval in experience_dependency_dict[key]: #Looping over webmaps corresponding to each experience
            innermost_dict = {}
            match = re.search(r'ID:\s*([\w]+)', webmapval, re.IGNORECASE) #Uses regex matching to extract webmap itemid from the dictionary value.
            idtext = match.group(1)
            mapItem = gis.content.get(idtext) #Finds webmap associated with the id value
            dependencies = mapItem.get_data(try_json = True)['operationalLayers'] #returns a dictionary with details on supporting services. try_json is optional but may help convert the data to a dictionary.
        
            for depinfo in dependencies: #Looping through the dependency JSON for a single web map
                dep_info_list = dependency_recursion(depinfo) #Pulls supporting services out of dependency JSON and adds to list
            

                # Matches the service feature class dependencies to the feature class dependency dict made from the input csv.
                # For every service that matches a url in the csv, the supporting feature classes are added as values.
                # This creates the innermost dictionary in the nested dictionary that will be inputted into pyvis.

                for each in dep_info_list: #Looping through the services associated with a single web map
                    match_found = False
                    for x, y in fc_dict.items(): 
                        if x in each:
                            innermost_dict[each] = y
                            match_found = True
                            break
                    if not match_found:
                        innermost_dict[each] = ["This Service is Hosted on AGOL or is a Raster Image"]
                                                                            
            inner_dict[f"{mapItem.title}, Item ID: {mapItem.id}"] = innermost_dict
            
        exp_dependency_dict[key] = inner_dict #writes keys and values to empty dictionary.
    except Exception as e:
        print(f'{key} {e}')
```


```python
# Accounting for edge cases such as maps with no services, experiences with no maps, etc. 

for webmap, service in map_dependency_dict.items():
    if not service:        
        map_dependency_dict[webmap] = {"This Webmap has no Supporting Services": ["No Services Therefore no FCs"]}
for exp, webmap in exp_dependency_dict.items():
    if not webmap:
        exp_dependency_dict[exp] = {"This Experience has no Supporting Services" : {"Likely no Webmap, Likely no Services": ["Likely no Services, Likely no FCs"]}}
for exp, webmap in exp_dependency_dict.items():       
    for wmap, service in webmap.items():
        if not service: 
            webmap[wmap] = {"This Webmap has no Supporting Services": ["No Services Therefore no FCs"]}
        

```


```python
# Creation of dashboard dependency dictionaries. Splits dependencies into flat and heiarchical structures e.g. {dashboard : service} and
# {dashboard : {webmap: {service}}}

dashboard_map_dependency_dict = {}
dashboard_service_dependency_dict = {}


for key in dashdict:
    temp_map_list = []
    temp_service_list = []
    try:
        dashboard = gis.content.get(dashdict[key])
        func_in = dashboard.get_data()
        final = find_web_maps(func_in)
        for each in final:
            if "Web Map" in each:
                temp_map_list.append(each)
                dashboard_map_dependency_dict[key] = temp_map_list
            else: 
                temp_service_list.append(each)
                dashboard_service_dependency_dict[key] = temp_service_list
    except Exception as e:
        print(f'{key}{e}')
```


```python
# See analogous experience code for notes

dash_dependency_dict = {}
for key in dashboard_map_dependency_dict: #Looping over experiences
    inner_dict = {}

    try:
        for webmapval in dashboard_map_dependency_dict[key]: #Looping over webmaps corresponding to each experience
            innermost_dict = {}
            match = re.search(r'ID:\s*([\w]+)', webmapval, re.IGNORECASE) #Uses regex matching to extract webmap itemid from the dictionary value.
            idtext = match.group(1)
            mapItem = gis.content.get(idtext) #Finds webmap associated with the id value
            dependencies = mapItem.get_data(try_json = True)['operationalLayers'] #returns a dictionary with details on supporting services. try_json is optional but may help convert the data to a dictionary.              
            dep_info_list = dependency_recursion(dependencies) #Pulls supporting services out of dependency JSON and adds to list

            for each in dep_info_list: #Looping through the services associated with a single web map
                match_found = False
                for x, y in fc_dict.items(): 
                    if x in each:
                        innermost_dict[each] = y
                        match_found = True
                        break
                if not match_found:
                    innermost_dict[each] = ["This Service is Hosted on AGOL or is a Raster Image"]
                                                                            
            inner_dict[f"{mapItem.title}, Item ID: {mapItem.id}"] = innermost_dict
            
        dash_dependency_dict[key] = inner_dict #writes keys and values to empty dictionary.
    except Exception as e:
        print(f'{key} {e}')
```


```python
dash_service_dict = {}
temp_dash_service_dict = {}
final_dash_service_dict = {}

for dash, webmap in dash_dependency_dict.items():
    for wmap, service in webmap.items():
        dash_service_dict[dash] = service

for dash, service, in dashboard_service_dependency_dict.items():
    for serv in service:
        temp_dash_service_dict[dash] = {serv : ['This Service is Hosted on AGOL or is a Raster Image']}

for dashboard, service in dash_service_dict.items():
    final_dash_service_dict[dashboard] = service
    for dash, serv in temp_dash_service_dict.items():
        if dash not in final_dash_service_dict.keys():
            final_dash_service_dict[dash] = serv

            
for dashboard, service in dash_service_dict.items():
    final_dash_service_dict[dashboard] = service
    for dash, serv in temp_dash_service_dict.items():        
        if dashboard == dash:
            service.update(serv)       

```


```python
# Creation of dependency matrix pandas dataframe from dependency dicts

empty_df = pd.DataFrame(dtype = "object") #dtype must be object to avoid errors

# for every service and fc, creates an index value if not seen before and creates an "x" at their intersection. If seen, just creates "x" at intersection.
for webmap, service in map_dependency_dict.items(): 
    if service:
        for serv, fc in service.items():
            empty_df.loc[serv, fc] = "x" 
empty_df = empty_df.replace(b"", np.nan) #removes blank anomalies
serv_fc_df = empty_df.copy() #consolidates the cells within memory
serv_fc_df_t = serv_fc_df.transpose()



empty_df = pd.DataFrame(dtype = "object")

for webmap, service in map_dependency_dict.items():
    if service:
        for serv, fc in service.items():
            empty_df.loc[webmap, fc] = "x"        
empty_df = empty_df.replace(b"", np.nan)
map_fc_df = empty_df.copy()
map_fc_df_t = map_fc_df.transpose()



empty_df = pd.DataFrame(dtype = "object")

for exp, webmap in exp_dependency_dict.items():
    if webmap:
        for wmap, service in webmap.items():
            for serv, fc in service.items():
                empty_df.loc[exp, fc] = "x"        
empty_df = empty_df.replace(b"", np.nan)
exp_fc_df = empty_df.copy()
exp_fc_df_t = exp_fc_df.transpose()

empty_df = pd.DataFrame(dtype = "object")

for dash, webmap in dash_dependency_dict.items():
    for wmap, service in webmap.items():
        for serv, fc in service.items():
            empty_df.loc[dash, fc] = "x"        
empty_df = empty_df.replace(b"", np.nan)
dash_fc_df = empty_df.copy()
dash_fc_df_t = dash_fc_df.transpose()
```


```python
writer = pd.ExcelWriter(fr"filepath\AGOL_Dependencies_{formatted_date_mdy}\Feature Class Dependencies.xlsx", 
    engine='xlsxwriter')

sheet1 = create_new_sheet(writer, serv_fc_df, "Services by FC", "Services")
sheet2 = create_new_sheet(writer, serv_fc_df_t, "FCs by Services", "Feature Classes")
sheet3 = create_new_sheet(writer, map_fc_df, "Webmaps by FC", "Webmaps")
sheet4 = create_new_sheet(writer, map_fc_df_t, "FCs by Webmap", "Feature Classes")
sheet5 = create_new_sheet(writer, exp_fc_df, "Experiences by FC", "Experiences")
sheet6 = create_new_sheet(writer, exp_fc_df_t, "FCs by Experience", "Feature Classes")
sheet7 = create_new_sheet(writer, dash_fc_df, "Dashboards by FC", "Experiences")
sheet8 = create_new_sheet(writer, dash_fc_df_t, "FCs by Dashboard", "Feature Classes")


writer.close()
```


```python
empty_df = pd.DataFrame(dtype = "object")

for webmap, service in map_dependency_dict.items():
    if service:
        for serv, fc in service.items():
            empty_df.loc[webmap, serv] = "x"
empty_df = empty_df.replace(b"", np.nan)
map_serv_df = empty_df.copy()
map_serv_df_t = map_serv_df.transpose()


empty_df = pd.DataFrame(dtype = "object")

for exp, webmap in exp_dependency_dict.items():
    if webmap:
        for wmap, service in webmap.items():
            if service:
                for serv, fc in service.items():
                    empty_df.loc[exp, serv] = "x"        
empty_df = empty_df.replace(b"", np.nan)
exp_serv_df = empty_df.copy()
exp_serv_df_t = exp_serv_df.transpose()

empty_df = pd.DataFrame(dtype = "object")

for dash, service in final_dash_service_dict.items():
    for serv, fc in service.items():
            empty_df.loc[dash, serv] = "x"        
empty_df = empty_df.replace(b"", np.nan)
dash_serv_df = empty_df.copy()
dash_serv_df_t = dash_serv_df.transpose()
```


```python
writer = pd.ExcelWriter(fr"filepath\AGOL_Dependencies_{formatted_date_mdy}\Feature Service Dependencies.xlsx", 
    engine='xlsxwriter')

sheet1 = create_new_sheet(writer, map_serv_df, "Webmaps by Service", "Webmaps")
sheet2 = create_new_sheet(writer, map_serv_df_t, "Services by Webmap", "Services")
sheet3 = create_new_sheet(writer, exp_serv_df, "Experiences by Service", "Experiences")
sheet4 = create_new_sheet(writer, exp_serv_df_t, "Services by Experience", "Services")
sheet5 = create_new_sheet(writer, dash_serv_df, "Dashboards by Service", "Dashboards")
sheet6 = create_new_sheet(writer, dash_serv_df_t, "Services by Dashboard", "Services")

writer.close()
```


```python
empty_df = pd.DataFrame(dtype = "object")

for exp, webmap in exp_dependency_dict.items():
    if webmap:
        for wmap, service in webmap.items():
            empty_df.loc[exp, wmap] = "x"        
empty_df = empty_df.replace(b"", np.nan)
exp_map_df = empty_df.copy()
exp_map_df_t = exp_map_df.transpose()

empty_df = pd.DataFrame(dtype = "object")

for dash, webmap in dash_dependency_dict.items():
    for wmap, service in webmap.items():
            empty_df.loc[dash, wmap] = "x"        
empty_df = empty_df.replace(b"", np.nan)
dash_map_df = empty_df.copy()
dash_map_df_t = dash_map_df.transpose()
```


```python
writer = pd.ExcelWriter(fr"filepath\AGOL_Dependencies_{formatted_date_mdy}\Webmap Dependencies.xlsx", 
    engine='xlsxwriter')

sheet1 = create_new_sheet(writer, exp_map_df, "Experiences by Webmap", "Experiences")
sheet2 = create_new_sheet(writer, exp_map_df_t, "Webmaps by Experience", "Webmaps")
sheet3 = create_new_sheet(writer, dash_map_df, "Dashboards by Webmap", "Dashboards")
sheet4 = create_new_sheet(writer, dash_map_df_t, "Webmaps by Dashboards", "Webmaps")

writer.close()
```


```python
# Creates a an Excel sheet showing feature, map, and vector tile services that are not consumed by webmaps, experiences, or dashboards.

orphan_services = [] 
temp_orphan_list = []
duplicate_list =[]
duplicate_list2 = []
for each in servid: # Ruling out services consumed by webmaps.
   if all(each not in serv for webmap, service in map_dependency_dict.items() for serv, fc in service.items()):
        duplicate_list.append(each)

for each in duplicate_list: # Ruling out services that aren't consumed by webmaps but are consumed by dashbaords.
    if all(each not in serv for dashboard, service in dashboard_service_dependency_dict.items() for serv in service):
        duplicate_list2.append(each)
        
for each in duplicate_list2: # Removes duplicates.
    if each not in orphan_services:
        temp_orphan_list.append(each)
        
for each in services_list:
    if any(servid in each for servid in temp_orphan_list):
        if "cityworks" not in each.lower():
            orphan_services.append(each)  

orphan_df = pd.DataFrame(orphan_services)

writer = pd.ExcelWriter(fr"filepath\AGOL_Dependencies_{formatted_date_mdy}\Orphan Services.xlsx", 
    engine='xlsxwriter')
orphan_df.to_excel(writer, sheet_name = "Sheet_1")

writer.close()
```


```python

```
