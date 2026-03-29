AGOL Dependency Automator

Purpose: 
        This script is designed to automate almost all the work involved in creating a comprehensive data lineage for AGOL content.
    Ouput data is visualized in a series of Excel sheets formatted as sortable dependency matrices. 

Method:
        There are two avenues of data capture in this script.    
    First, it compiles a dictionary of all webmaps in AGOL and parses their related JSONs to extract the services they depend upon. 
    Those services are then matched to a dictionary of feature services and feature classes, producing a dictionary structured like
    {webmap : {service : [feature classes]}}. This is then transformed into a pandas dataframe, transposed, and then written as a 
    formatted Excel sheet. Second, the same process is performed on a dictionary of experiences, creating a dictionary structured as 
    {experience : {webmap : {service : [feature classes]}}.

Inputs:
        This script takes inputs from two sources. The first is directly from Portal/AGOL via the ArcGIS API, and the second is a 
    manually created document detailing which feature classes services consume. It may be possible to automate the association of 
    feature classes to registered services by renaming .aprx files into .zip files and programatically inspecting the map JSONS within. 
        


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

expitems = []
exptitle = []
expid = []
maptitle = []
mapid = []
mapitems = []
dashtitle = []
dashid = []
dashitems = []

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
dashboards = gis.content.search(query = "type:Dashboard", item_type = "Dashboard", max_items = 1000)

if web_exps:
    for web_exp in web_exps:
        exptitle.append(web_exp.title)
        expid.append(web_exp.id)
        expitems.append(web_exp)
        
tempitemdictexp = dict(zip(expid, expitems))

if web_maps:
    for web_map in web_maps:
        maptitle.append(web_map.title)
        mapid.append(web_map.id) 
        mapitems.append(web_map)
        
tempitemdictmap = dict(zip(mapid, mapitems))
                         
if dashboards:
    for dashboard in dashboards:
        dashtitle.append(dashboard.title)
        dashid.append(dashboard.id)
        dashitems.append(dashboard)

tempitemdictdash = dict(zip(dashid, dashitems))
        
        
expdict = dict(zip(exptitle, expid)) 
mapdict = dict(zip(maptitle, mapid)) 
dashdict = dict(zip(dashtitle, dashid))
portal_item_dict = tempitemdictexp | tempitemdictdash | tempitemdictmap
```


```python
dependency_df = pd.read_csv(r"filepath\featureclassmapping.csv")

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
        if "itemId" in data:
            dep_id = data.get('itemId')
            if dep_id is not None:
                dep_item = portal_item_dict.get(dep_id)
                if dep_item:
                    info_str = info_str = (f"{dep_item.title}, Item ID: {dep_item.id}, {dep_item.type}")
                    found_values.append(info_str)
                else:
                    print(f'Service {dep_id} does not exist')
                    
        for key, value in data.items():
            find_web_maps(value, found_values)
            
    elif isinstance(data, list): 
        for each in data:
            find_web_maps(each, found_values)
    
    return found_values


# Recursively searches though map dependency JSON for item IDs and rest service URLs and creates a list of dependencies.
# Used within a for loop of web maps.

def dependency_recursion(data,found_values=None):
    if found_values is None:
        found_values = []

    if isinstance(data, dict):
        if 'itemId' in data:
            dep_id = data.get('itemId')
            dep_item = portal_item_dict.get(dep_id)
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
    deptemp = portal_item_dict.get(mapdict[each])
    infostrtemp = (f"{deptemp.title}, Item ID: {deptemp.id}")
    templist.append(infostrtemp)
    mapdict2[infostrtemp] = mapdict[each]
```


```python
map_dependency_dict = {}
for key in mapdict2:
    innermost_dict = {}
    try:
        mapItem = portal_item_dict.get(mapdict2[key]) #finds webmap associated with the id value in mapdict.
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
        experience = portal_item_dict.get(expdict[key])
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
            mapItem = portal_item_dict.get(idtext) #Finds webmap associated with the id value
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
        dashboard = portal_item_dict.get(dashdict[key])
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
            mapItem = portal_item_dict.get(idtext) #Finds webmap associated with the id value
            dependencies = mapItem.dependent_upon()['list'] #returns a dictionary with details on supporting services. try_json is optional but may help convert the data to a dictionary.              
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
# Could be replaced with the pandas crosstab function?

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
