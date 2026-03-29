
**Solving complicated problems and creating reliable, low-maintenance systems, like with this AGOL Depenency Automator Script**

About:
This tool was inspired by a workflow developed during my time at the City of Billings, rewritten and heavily pared down for general public use with permission.

Purpose: 
This script automates the creation of dependency lineage documentation for AGOL Web Experiences. While this specific script only details the flow of data from services into webmaps into Experiences, it can be expanded to cover all AGOL/Portal applications and dig down to the feature class level, allowing the mapping of feature classes to user-facing applications. 

Method:
This script compiles a dictionary of all webmaps in AGOL and parses their related JSONs to extract the services they depend upon and then creates a dictionary structured as {experience : {webmap : [services]}. This is converted into a Pandas dataframe and then written to a formatted excel sheet. 
        


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

if web_exps:
    for web_exp in web_exps:
        exptitle.append(web_exp.title)
        expid.append(web_exp.id)
        expitems.append(web_exp)
        
tempitemdictexp = dict(zip(expid, expitems))
              
expdict = dict(zip(exptitle, expid)) 

portal_item_dict = tempitemdictexp | tempitemdictdash | tempitemdictmap
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
                                                                                    
        exp_dependency_dict[key] = dep_info_list #writes keys and values to empty dictionary.
        
    except Exception as e:
        print(f'{key} {e}')
```


```python
# Accounting for edge cases such as maps with no services, experiences with no maps, etc. 
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
# Creation of dependency matrix pandas dataframe from dependency dicts
# Could be replaced with the pandas crosstab function


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

for exp, webmap in exp_dependency_dict.items():
    if webmap:
        for wmap, service in webmap.items():
            empty_df.loc[exp, wmap] = "x"        
empty_df = empty_df.replace(b"", np.nan)
exp_map_df = empty_df.copy()
exp_map_df_t = exp_map_df.transpose()


```


```python
writer = pd.ExcelWriter(fr"filepath\AGOL_Dependencies_{formatted_date_mdy}\Feature Class Dependencies.xlsx", 
    engine='xlsxwriter')

sheet5 = create_new_sheet(writer, exp_fc_df, "Experiences by FC", "Experiences")
sheet6 = create_new_sheet(writer, exp_fc_df_t, "FCs by Experience", "Feature Classes")

writer.close()
```


```python
writer = pd.ExcelWriter(fr"filepath\AGOL_Dependencies_{formatted_date_mdy}\Feature Service Dependencies.xlsx", 
    engine='xlsxwriter')

sheet3 = create_new_sheet(writer, exp_serv_df, "Experiences by Service", "Experiences")
sheet4 = create_new_sheet(writer, exp_serv_df_t, "Services by Experience", "Services")
```


```python
writer = pd.ExcelWriter(fr"filepath\AGOL_Dependencies_{formatted_date_mdy}\Webmap Dependencies.xlsx", 
    engine='xlsxwriter')

sheet1 = create_new_sheet(writer, exp_map_df, "Experiences by Webmap", "Experiences")
sheet2 = create_new_sheet(writer, exp_map_df_t, "Webmaps by Experience", "Webmaps")

writer.close()
```
