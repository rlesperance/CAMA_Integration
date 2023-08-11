## Notebook Code

This code takes a slightly different angle than the file Geodatabase overwrite.  Though the python scripts can have a combination of methods.  
For instance you could compile the file based (shapefile/excel) data into the final file geodatabase and truncate and append the data instead of upload the file geodatabase and overwrite the service. 
This may preserve some functionality of the service, such as popup or symbology that has been set at the item level.  

This script...
1. identifies the feature class and CAMA table, assuming they are on an Enterprise Geodatabase
2. Creates a database query joining them together
3. Exports the data to a new layer
4. Uses the delete and append tools in ArcGIS

## Set up code

```python
import arcpy
from arcgis.gis import GIS
from arcgis import features
import traceback, datetime, os, sys
import logging
import concurrent
import time
import requests

logging.basicConfig(filename = r".\Update_Parcels_Log.txt",level=logging.INFO)
logging.info("{} Begin Moving Parcels ".format(str(datetime.datetime.now()))) 

arcpy.env.overwriteOutput = True
arcpy.env.qualifiedFieldNames = False

#GET SOURCE FEATURE CLASS
basePath = r"C:\ProjectFolder"
SDE = os.path.join(basePath, "sqlserver_GISDB.sde")
parcelClass = os.path.join(SDE, "data.Parcels")
CAMA = os.path.join(SDE, "data.CAMA")

tempParcels = os.path.join(SDE, "Temp_Lucas_Parcels")

#GET ArcGIS ONLINE FEATURE LAYER
gis = GIS("https://agency.maps.arcgis.com", "username", "password")
parcelService = gis.content.get("e058c4c8963b431a8a2e321fa82b41cd")
```

## Functions

REINDEXFS - appending data should re-index the tables, but lets kick that refrigerator
You should look at the feature service's definition to get the exact names of the indexes

```python

def reindexFS(fLayer):

    sp_index = {"indexes" : [
        {"name" : "user_12345.PARCELS_CAMA_Shape_sidx", "fields" : "Shape"}
      ]}
    indexes_dict = {"indexes" : [
        {"name" : "PK__PARCELS__F4B70D85165D6111", "fields" : "OBJECTID"}, 
        {"name" : "ParcelID_Index", "fields" : "ParcelID"}, 
        {"name" : "FDO_GlobalID", "fields" : "GlobalID"}
      ]}
    
    try:
        result = fLayer.manager.update_definition(sp_index)
        print (result)
        logging.info("Spatial Index result:  {}".format(result))
        
        result = fLayer.manager.update_definition(indexes_dict)
        print (result)
        logging.info("Index result:  {}".format(result))
        
    except Exception as err:
        print(err)
        logging.info("Index result:  {}".format(result))
        logging.error (str(sys.exc_info()) + "\n")
        logging.error (traceback.format_tb(sys.exc_info()[2])[0] + "\n")

```

## Main block

This main block assumes there is an Enterprise GDB, and makes use of the MakeQueryLayer tool. 

```python
try:
    startTime = time.localtime(time.time())
    print (str(startTime.tm_hour) + ":" + str(startTime.tm_min) + ":" + str(startTime.tm_sec))

    #CAMA
    print ("Create CAMA query layer")
    queryString = "select a.objectid, a.ParcelID, a.GlobalID, a.shape, b.ownername, b.address, b.city, b.state, b.zip, b.lotSqFt "
    queryString += "from data.Parcels a inner join data.CAMA b on a.ParcelID=b.CamaID"
    qLayer = arcpy.management.MakeQueryLayer(SDE, "ParcelsCama", queryString, "OBJECTID")

    tmpParcels = arcpy.CopyFeatures_management(qLayer, tempParcels)

    parcelLayer = parcelService.layers[0]

    print ("Appending parcel features")
    if len(featureset) > 0:
        result = arcpy.management.TruncateTable(parcelLayer._url)
        print ("Truncate:  {}".format(result))

        result = arcpy.management.Append(tmpParcels, parcelLayer._url)  #assumes a field to field match.  See file GDB method for append field mapping
        print ("Append:  {}".format(result))
    
    print("re-Indexing parcel layer")
    reindexFS(parcelLayer)

except Exception as err:
    print(err)
    logging.error (str(sys.exc_info()) + "\n")
    logging.error (traceback.format_tb(sys.exc_info()[2])[0] + "\n")

finishTime = time.localtime(time.time())
print (str(finishTime.tm_hour) + ":" + str(finishTime.tm_min) + ":" + str(finishTime.tm_sec))

print("Finish")
logging.info("{} Finished! ".format(str(datetime.datetime.now())))
```
