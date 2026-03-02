AGOL Dependency Automator

Purpose: 
        This script is designed to automate almost all the work involved in creating a comprehensive data lineage for AGOL content.
    Moreoever, this script also documents feature services that are not consumed by webmaps for the purposes of data governance. 
    Ouput data is visualized in a series of Excel sheets formatted as sortable dependency matrices. 

Method:
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

Documentation location: "\\PWU-W12k07\GIS_Files\Tutorials\Dependencies\Creating Input Data for Dependency Visualizer.docx"

        

    
        


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
#os.mkdir(fr"\\PWU-W12k07\GIS_Files\Tutorials\Dependencies\AGOL\AGOL_Dependencies_{formatted_date_mdy}")

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

portal_url = "https://billings.maps.arcgis.com"  
username = "GisAdminAcct"
password = "GAdmin25@2024"     

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

    Successfully connected to ArcGIS Online
    


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
dependency_df = pd.read_csv(r"\\PWU-W12k07\GIS_Files\Tutorials\Dependencies\FeatureServiceFeatureClassDependencies.csv")

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

    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 33632d6ab9cd4fd8b6fb09d42cf2420f does not exist
    Service 7c26778aeb7a49cbaeb39288af899117 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service f45ec5d48dfd458dafcedce5ee97d61d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 03a50e6d46f3411184f701695e3decc2 does not exist
    US Population Change, Item ID: e634e840f62d49d18c14938926ac6041 You do not have permissions to access this resource or perform this operation.
    (Error Code: 403)
    Service 56e9dd833fbf4c9dae3cbfee575c7cb6 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 7c26778aeb7a49cbaeb39288af899117 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service f45ec5d48dfd458dafcedce5ee97d61d does not exist
    Service 33632d6ab9cd4fd8b6fb09d42cf2420f does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 7c26778aeb7a49cbaeb39288af899117 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service f45ec5d48dfd458dafcedce5ee97d61d does not exist
    Service 7326058bdcc54002a5c13d02cb41b3ea does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 7c26778aeb7a49cbaeb39288af899117 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service f45ec5d48dfd458dafcedce5ee97d61d does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service 7c26778aeb7a49cbaeb39288af899117 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service f45ec5d48dfd458dafcedce5ee97d61d does not exist
    Service 7c26778aeb7a49cbaeb39288af899117 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service ed17a08d5d8b4a5d9ca29c02449ac1d5 does not exist
    Service f45ec5d48dfd458dafcedce5ee97d61d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service b059f0ac7d2b4ee8afb4b03105b1e6d1 does not exist
    Service b059f0ac7d2b4ee8afb4b03105b1e6d1 does not exist
    


```python

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

    Untitled experience 1 You do not have permissions to access this resource or perform this operation.
    (Error Code: 403)
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 7326058bdcc54002a5c13d02cb41b3ea does not exist
    Service 7326058bdcc54002a5c13d02cb41b3ea does not exist
    Service 7326058bdcc54002a5c13d02cb41b3ea does not exist
    Service 7326058bdcc54002a5c13d02cb41b3ea does not exist
    Service 7326058bdcc54002a5c13d02cb41b3ea does not exist
    Service 359b82da4d1342e58190fab4d814530b does not exist
    Service 359b82da4d1342e58190fab4d814530b does not exist
    Service 359b82da4d1342e58190fab4d814530b does not exist
    Service 359b82da4d1342e58190fab4d814530b does not exist
    Service 359b82da4d1342e58190fab4d814530b does not exist
    Service 359b82da4d1342e58190fab4d814530b does not exist
    Service 359b82da4d1342e58190fab4d814530b does not exist
    Service 359b82da4d1342e58190fab4d814530b does not exist
    Service 359b82da4d1342e58190fab4d814530b does not exist
    Service 359b82da4d1342e58190fab4d814530b does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service b999b316d1b0425383f161ff0e21142d does not exist
    


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
        final = dash_find_web_maps(func_in)
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

    Service 4af68b42-c6ab-4aaa-a022-57cbe36e6cbc does not exist
    Service 4af68b42-c6ab-4aaa-a022-57cbe36e6cbc does not exist
    Service 886981f7-3d6b-4cab-aee2-5a905c54df47 does not exist
    Service e8d81e2a-86aa-4435-bab6-f7d7eda13457 does not exist
    Service 3efab53cc9ea485085f74747af7f079a does not exist
    Service 886981f7-3d6b-4cab-aee2-5a905c54df47 does not exist
    Service e8d81e2a-86aa-4435-bab6-f7d7eda13457 does not exist
    Service 3efab53cc9ea485085f74747af7f079a does not exist
    Service 4c132d54-8229-4921-9fde-b95fc27d60d9 does not exist
    Service 77a07dcfb6e947c093fc2bba3c375fc2 does not exist
    Service 4c132d54-8229-4921-9fde-b95fc27d60d9 does not exist
    Service 4c132d54-8229-4921-9fde-b95fc27d60d9 does not exist
    Service 77a07dcfb6e947c093fc2bba3c375fc2 does not exist
    Service 4c132d54-8229-4921-9fde-b95fc27d60d9 does not exist
    Service 4c132d54-8229-4921-9fde-b95fc27d60d9 does not exist
    Service 4c132d54-8229-4921-9fde-b95fc27d60d9 does not exist
    Service 77a07dcfb6e947c093fc2bba3c375fc2 does not exist
    Service 77a07dcfb6e947c093fc2bba3c375fc2 does not exist
    Service ae384b06-dec9-4758-bafe-01d9d79a4a1f does not exist
    Service 54da2a15-0acc-4cae-8d85-ab8aaef68c21 does not exist
    Service 081d58dd2620479d8031b1089c5a024e does not exist
    Service 081d58dd2620479d8031b1089c5a024e does not exist
    Service 081d58dd2620479d8031b1089c5a024e does not exist
    Service ae384b06-dec9-4758-bafe-01d9d79a4a1f does not exist
    Service 54da2a15-0acc-4cae-8d85-ab8aaef68c21 does not exist
    Service 081d58dd2620479d8031b1089c5a024e does not exist
    Service 54da2a15-0acc-4cae-8d85-ab8aaef68c21 does not exist
    Service 081d58dd2620479d8031b1089c5a024e does not exist
    Service 081d58dd2620479d8031b1089c5a024e does not exist
    Service 081d58dd2620479d8031b1089c5a024e does not exist
    Service 081d58dd2620479d8031b1089c5a024e does not exist
    Service 081d58dd2620479d8031b1089c5a024e does not exist
    Service 081d58dd2620479d8031b1089c5a024e does not exist
    Service f395068006db4c19b4e533024701adc5 does not exist
    Service 61d987d70c604fcd8bb2f4ccaa9b8974 does not exist
    Service 61d987d70c604fcd8bb2f4ccaa9b8974 does not exist
    Service 61d987d70c604fcd8bb2f4ccaa9b8974 does not exist
    Service 61d987d70c604fcd8bb2f4ccaa9b8974 does not exist
    


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

    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 6df8c73e09624e028c5840a1303d5062 does not exist
    Service 33632d6ab9cd4fd8b6fb09d42cf2420f does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service d1739d21274847e8a2d25c90100375c3 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    Service 9387cf9b99c844c4a56f59417e5223b4 does not exist
    


```python
dash_dependency_dict
```




    {'Citizen Problem Dashboard': {'Citizen Problem Dashboard, Item ID: fae2292874ba46ad939548bdb74021d8': {'Requests || 5b7e46a546d44a4c969edd58fa8bb951 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Requests_7352434fce2d45bc9ad12343e052b44e/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']},
      'Citizen Problem Survey, Item ID: 023e93827d354abd93df4ae94e8430c9': {'Requests_survey || 950f8eae79cf471e877e25840c2d0240 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Requests_survey_7352434fce2d45bc9ad12343e052b44e/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Snowplow Dashboard Base - BJ': {'Snow Response Map, Item ID: 5da6bcd872124b59b45d36bfa9c89095': {'Snow Response Map_WFL1 || 998628deb4b144e8a17afd9ab606d700 || Feature Service || https://services.arcgis.com/ue9rwulIoeLEI9bj/arcgis/rest/services/Snow_Response_Map_WFL1/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Engineering Dashboard': {'Engineering Map_RO, Item ID: a1356197bc774e788d7a03454c476b9f': {'Transportation || c33a2c057a884048a7398023224e7dde || Feature Service || https://billingsgis.com/maps/rest/services/MapServices_HDR/Transportation/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'GIS_Sewer_Utility || 5f1eda38a8994e5aac1cb30b3ee609c8 || Feature Service || https://utility.arcgis.com/usrsvcs/servers/5f1eda38a8994e5aac1cb30b3ee609c8/rest/services/MapServices_HDR/GIS_Sewer_Utility/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Life Expectancy Sewer || 94df6241520d46e19949d64e3ca34bb3 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Life_Expectancy_Sewer_WFL1/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'BPD Offenses DB': {'BPDOffensesMap, Item ID: abf0a2cb62d04c288e0faaf7af2f690e': {'bpd_offenses || 4d221c31409e474f8c7313de566d9154 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/bpd_offenses/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Snowplow Dashboard Base Exercise 2 - BJ': {'Snow Response Map, Item ID: 5da6bcd872124b59b45d36bfa9c89095': {'Snow Response Map_WFL1 || 998628deb4b144e8a17afd9ab606d700 || Feature Service || https://services.arcgis.com/ue9rwulIoeLEI9bj/arcgis/rest/services/Snow_Response_Map_WFL1/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Parks Classification Map Dashboard': {'Park Classification Map, Item ID: ec035b4a2a33478ab175194d9dea10ef': {'https://billingsgis.com/arcgis_public/rest/services/ArcOnline_Public/Park_Classification_EXT/Mapserver': ['Feature Class: CityLimits | Feature Dataset: AdministrativeArea',
        'Feature Class: Billings_Parks | Feature Dataset: Parks_Rec']}},
     'Sober Housing Dashboard': {'Sober Housing Web Map, Item ID: 99d2853dce8446bc9c7d17d3323abc58': {'shousing_nearby_psn || 6773e7ff4eec44579fe53ea7b5c80cbb || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/shousing_nearby_psn/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'sober_housing_buffer_250_ft || 0e3bb04378134edda6674cf0e2c3a0c8 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/sober_housing_buffer_250_ft/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'psn_data_join_housing_buffer || 7a0795dcd30a49bb90cc1e8d4de1995e || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/psn_data_join_housing_buffer/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'psnData || 465d7b11fad74fb5a7766278ae0ea560 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/psnData/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'RedSphere.png': ['This Service is Hosted on AGOL or is a Raster Image'],
       'psnDrugs || f93e2ea0cc4848ea991f60953bf84c3e || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/psnDrugs/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Billings Capital Project Dashboard (copy)': {'Capital Project Dashboard, Item ID: 1157014d0e5e476a891052f2e4b57ce4': {'Billing TIF Districts || 4bf13e117d424bc9a32b0e9d127b484c || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billing_TIF_Districts/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Billings Wards || a31aa0fe18f94592819456154b5bc763 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billings_Wards/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'InfrastructureProjects_allfundedprojects || 8fb969776ad54bfe87b5c8d13f8ae007 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/InfrastructureProjects_allfundedprojects_e560caa0cc4740f1823df06287488dba/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Tree Dashboard': {'Tree Dashboard, Item ID: 1e0f809a127d4ce7bbff2d51ad4244dc': {'Trees || 4404bdd533a341c6be868aec4c600295 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Trees_0a2fe9e754d14e5a8ca091c49f47278f/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Trees_currentinspection || ba678ce0aa3c4496aa8d1eb6a099d813 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Trees_currentinspection_0a2fe9e754d14e5a8ca091c49f47278f/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Demo Crime Dashboard - Interactive': {'Philadelphia Crime Map, Item ID: 7dd95c7265bd40cba1fd36626f43c5dc': {'Demo Philadelphia Crime || e63a5d3be52b466eac6a75ed6e255087 || Feature Service || https://services.arcgis.com/P3ePLMYs2RVChkJx/arcgis/rest/services/Philadelphia_Crime_Map_WFL1/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://services.arcgis.com/P3ePLMYs2RVChkJx/arcgis/rest/services/Philadelphia_Crime_Map_WFL1/FeatureServer/0': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/3f3435eaf4914c3f8224ccd4c25402b0/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/9cf57dfa85a648c49ac8a274fd45dd24/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/0d123a1b7078425ab75fed5c1cb179de/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/067f95fb264b4aa9ae7b40bbcf5db695/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/86fa3216503f4c508796e9ffd242d8bd/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/b6e3304b4f5c4c568bc65e4636bda973/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/de4c5139f0eb491490af7274162a7059/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/c888b7e7af2f47448b6aa71a4cabc3ea/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/9b47202ed2964b2ebdb208a8f2750943/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/99802c02e0d443efb641f38a9151e355/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/82c06a5d2c744819b712ca99801ee8eb/data': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://www.arcgis.com/sharing/rest/content/items/38babc55ebc742b68b9b2390f6f006b6/data': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Capital Project Dashboard (new)': {'Capital Project Dashboard, Item ID: 1157014d0e5e476a891052f2e4b57ce4': {'Billing TIF Districts || 4bf13e117d424bc9a32b0e9d127b484c || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billing_TIF_Districts/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Billings Wards || a31aa0fe18f94592819456154b5bc763 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billings_Wards/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'InfrastructureProjects_allfundedprojects || 8fb969776ad54bfe87b5c8d13f8ae007 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/InfrastructureProjects_allfundedprojects_e560caa0cc4740f1823df06287488dba/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Billings Capital Project Dashboard': {'Capital Project Dashboard, Item ID: 1157014d0e5e476a891052f2e4b57ce4': {'Billing TIF Districts || 4bf13e117d424bc9a32b0e9d127b484c || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billing_TIF_Districts/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Billings Wards || a31aa0fe18f94592819456154b5bc763 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billings_Wards/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'InfrastructureProjects_allfundedprojects || 8fb969776ad54bfe87b5c8d13f8ae007 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/InfrastructureProjects_allfundedprojects_e560caa0cc4740f1823df06287488dba/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'CFS Analysis Dashboard': {'Buildings CFS, Item ID: 5088907de99f4626ae3aee4b7616cb1d': {'arc_data || a89f0858e15347adb25cbd36f0a4031a || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/arc_data/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Join_Features_to_billings_buildings || bc3d8f3cbe8e422abf49f8087f21fe00 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Join_Features_to_billings_buildings/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'PSN Overview Dashboard': {'PSN Data Map, Item ID: 7c2d2c9b224d4691997402ea3faeed2d': {'bpd_zones || cadf0281880640f48f0d8f15bbaf02bb || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/bpd_zones/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'psnData || 465d7b11fad74fb5a7766278ae0ea560 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/psnData/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'OLD - Billings MPO LRTP Projects Dashboard Map': {'Billings MPO LRTP Projects Dashboard Map, Item ID: 87767048e20c48328d7174baad1a51ad': {'BillingsMPO_LRTP_Projects_09032025 || d4594a5edde64b87971e41572125ddfc || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/BillingsMPO_LRTP_Projects_09032025/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Billings Planning Area Update || 561ea8a0ff7e46fa8ef2802187f0bebe || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billings_Planning_Area_Update/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'City of Billings- Solid Waste Cleanup': {'Public Solid Waste Cleanup Web Map, Item ID: 72b950d83f494fe39f1e9b0dfd6ecfca': {'Solid_Waste_Cleanup_Boundaries_view || 95b0ed60c7ee4a5fbdf17ffd4f34b0ee || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Solid_Waste_Cleanup_Boundaries_view/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Solid_Waste_Cleanup_Areas_view || 6fc979c6f52b45dabfbfce93b8c3a1f4 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Solid_Waste_Cleanup_Areas_view/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'City of Billings - Street Sweeping': {'Billings Street Sweeping Web Map Public, Item ID: 75562589a9ed41bc9af8a733f95c9852': {'Street Sweeping Public || c053afeb3bb94e16a7ec55eb64ac152a || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Street_Sweeping_Public/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Billings Capital Project Dashboard v2': {'Billings Capital Project Dashboard v2, Item ID: 707165ef23b2491ab2b84943ee59fe82': {'https://billingsgis.com/arcgis_public/rest/services/ArcOnline_Public/Election_Districts_EXT/MapServer/3': ['Feature Class: ACTVADDR | Feature Dataset: Address',
        'Feature Class: CityLimits_Dissolve | Feature Dataset: AdministrativeArea',
        'Feature Class: TaxParcels_YCO | Feature Dataset: ArcOnline_Data',
        'Feature Class: Schools_OPI | Feature Dataset: CityInfrastructure',
        'Feature Class: PollingPlacePolys | Feature Dataset: ElectionAdministration',
        'Feature Class: VotingPrecinct | Feature Dataset: ElectionAdministration',
        'Feature Class: Ward_Dissolve | Feature Dataset: ElectionAdministration',
        'Feature Class: City_Task_Force_Areas | Feature Dataset: LandUsePlanning',
        'Feature Class: Community_Planning_Neighborhoods | Feature Dataset: LandUsePlanning',
        'Feature Class: ZIPCODE | Feature Dataset: LandUsePlanning',
        'Feature Class: RoadCenterline | Feature Dataset: Transportation'],
       'https://it-cityhallgis.billings.ad:6443/arcgis/rest/services/ArcOnline_Public/Election_Districts_EXT/MapServer/3': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Billing TIF Districts || 4bf13e117d424bc9a32b0e9d127b484c || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billing_TIF_Districts/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'CIP_view || 58702131fe894e74b4a9a62ba0832aee || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/CIP_view/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Environmental Affairs Backflow Dashboard': {'Backflow Inspection Map, Item ID: 5975810699c24b9d9d34dd24d87702ed': {'Backflow Water Distribution Feature Layers || c32a917bbb6242468ddf2f73fd95ba02 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Backflow_Water_Distribution_Feature_Layers/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Backflow Prevention Device || 2f2d8cf9d7f84dc8aa7a015ca2fe9089 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Backflow_Prevention_Device/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Multi-Use Trails Map JG': {'Multi-Use Trails Map JG, Item ID: 44815f4ff20d433dbd612d7b41c14977': {'https://billingsgis.com/arcgis_public/rest/services/ArcOnline_Raster/ArcOnline_NAIP_Imagery/MapServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://billingsgis.com/arcgis_public/rest/services/ArcOnline_Raster/ArcOnline_2020_Imagery/MapServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'https://billingsgis.com/arcgis_public/rest/services/ArcOnline_Public/Multi_Use_Trails_EXT/MapServer': ['Feature Class: CityLimits_Dissolve | Feature Dataset: AdministrativeArea',
        'Feature Class: Trail_Map_Sites | Feature Dataset: Parks_Rec',
        'Feature Class: Bridge_Line | Feature Dataset: Transportation',
        'Feature Class: Multi_Use_Trails | Feature Dataset: Transportation']}},
     'PSN Firearms Dashboard': {'Firearm Crime Map Jan-April 2023, Item ID: 7c44c6e3fc2c486e8c53b4fe87f4523a': {'January_May_2023_Firearm_Data_Crunched_EXCEL TRUE || 753fefb9ccb248f1ae70f7093f1828f0 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/January_May_2023_Firearm_Data_Crunched_EXCEL_TRUE/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'TF Incidents DB': {'TF Incidents Map, Item ID: a1cb4b9acbb6413db62ca9cfa5d45c5a': {'tf_incidents || 73163d802afe48549fe85a840cd5c224 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/tf_incidents/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'General Fund Parks Map Dashboard': {'General Fund Parks, Item ID: 1dfb4ab647f244de804d091c36787b8b': {'https://billingsgis.com/arcgis_public/rest/services/ArcOnline_Public/General_Fund_Parks/Mapserver': ['Feature Class: CityLimits | Feature Dataset: AdministrativeArea',
        'Feature Class: Billings_Parks | Feature Dataset: Parks_Rec']}},
     'TF Areas Dashboard': {'TF Area Offenses, Item ID: d96461b44d5a443fac7788bfcb305d0c': {'tfoffenses_rolling_6months_online || f315bef7ab2a43a0a1cac3c1e677ddbe || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/tfoffenses_rolling_6months_online/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Capital Project Mobile Dashboard': {'Capital Project Dashboard, Item ID: 1157014d0e5e476a891052f2e4b57ce4': {'Billing TIF Districts || 4bf13e117d424bc9a32b0e9d127b484c || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billing_TIF_Districts/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Billings Wards || a31aa0fe18f94592819456154b5bc763 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billings_Wards/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'InfrastructureProjects_allfundedprojects || 8fb969776ad54bfe87b5c8d13f8ae007 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/InfrastructureProjects_allfundedprojects_e560caa0cc4740f1823df06287488dba/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'City of Billings - Traffic Counts': {'Traffic Count, Item ID: b1150ae483784827ba0bbd4b40c9ffc0': {'Traffic Count Public || cbda4de29b804c359763ae66ffb3c3e0 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Traffic_Count_Public/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'PSN Drug Dashboard': {'PSN Drug Map, Item ID: 64afed194a634e689665b2c998566cf8': {'psnDrugs || f93e2ea0cc4848ea991f60953bf84c3e || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/psnDrugs/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Planting Areas Dashboard': {'Planting Areas Dashboard, Item ID: dae94d6fa3c6476480f7e8cd747d51ec': {'Trees || 4404bdd533a341c6be868aec4c600295 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Trees_0a2fe9e754d14e5a8ca091c49f47278f/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'PlantingAreas_currentinspection || a76511b132604c969451387cc147dd30 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/PlantingAreas_currentinspection_0a2fe9e754d14e5a8ca091c49f47278f/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Tree Request Dashboard': {'Tree Request Dashboard, Item ID: 78208d72880a451b897133506d6f253a': {'Requests_tree || e55d0f3a45764741a41fc625eb09fd92 || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Requests_tree_0a2fe9e754d14e5a8ca091c49f47278f/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Billings MPO LRTP Projects Dashboard Map': {'Billings MPO LRTP Projects Dashboard Map, Item ID: 87767048e20c48328d7174baad1a51ad': {'BillingsMPO_LRTP_Projects_09032025 || d4594a5edde64b87971e41572125ddfc || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/BillingsMPO_LRTP_Projects_09032025/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image'],
       'Billings Planning Area Update || 561ea8a0ff7e46fa8ef2802187f0bebe || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Billings_Planning_Area_Update/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}},
     'Citizen Problem Explorer': {'Citizen Problem Explorer, Item ID: 9d77d6608c6b4ae5a9be638df2b2fff4': {'Requests_public || 2d589f35fbb34610b69722b73738fe6f || Feature Service || https://services6.arcgis.com/rCC3yWJa2mjYtKDP/arcgis/rest/services/Requests_public_00e63199176f44b788fd43684476713d/FeatureServer': ['This Service is Hosted on AGOL or is a Raster Image']}}}




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
writer = pd.ExcelWriter(fr"\\PWU-W12k07\GIS_Files\Tutorials\Dependencies\AGOL\AGOL_Dependencies_{formatted_date_mdy}\AGOL_Feature Class Dependencies_{formatted_date_mdy}.xlsx", 
    engine='xlsxwriter')

sheet1 = create_new_sheet(writer, serv_fc_df, "Services by FC", "Services")
sheet2 = create_new_sheet(writer, serv_fc_df_t, "FCs by Services", "Feature Classes")
sheet3 = create_new_sheet(writer, map_fc_df, "Webmaps by FC", "Webmaps")
sheet4 = create_new_sheet(writer, map_fc_df_t, "FCs by Webmap", "Feature Classes")
sheet5 = create_new_sheet(writer, exp_fc_df, "Experiences by FC", "Experiences")
sheet6 = create_new_sheet(writer, exp_fc_df_t, "FCs by Experience", "Feature Classes")
sheet7 = create_new_sheet(writer, dash_fc_df, "Dashboards by FC", "Dashboards")
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

    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    C:\Users\norellp\AppData\Local\Temp\ipykernel_22152\2159096078.py:6: PerformanceWarning: DataFrame is highly fragmented.  This is usually the result of calling `frame.insert` many times, which has poor performance.  Consider joining all columns at once using pd.concat(axis=1) instead. To get a de-fragmented frame, use `newframe = frame.copy()`
      empty_df.loc[webmap, serv] = "x"
    


```python
writer = pd.ExcelWriter(fr"\\PWU-W12k07\GIS_Files\Tutorials\Dependencies\AGOL\AGOL_Dependencies_{formatted_date_mdy}\AGOL_Feature Service Dependencies_{formatted_date_mdy}.xlsx", 
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
writer = pd.ExcelWriter(fr"\\PWU-W12k07\GIS_Files\Tutorials\Dependencies\AGOL\AGOL_Dependencies_{formatted_date_mdy}\AGOL_Webmap Dependencies_{formatted_date_mdy}.xlsx", 
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

writer = pd.ExcelWriter(fr"\\PWU-W12k07\GIS_Files\Tutorials\Dependencies\AGOL\AGOL_Dependencies_{formatted_date_mdy}\AGOL_Orphan Services_{formatted_date_mdy}.xlsx", 
    engine='xlsxwriter')
orphan_df.to_excel(writer, sheet_name = "Sheet_1")

writer.close()
```


```python

```
