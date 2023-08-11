## Notebook Code

Some agencies have very simple GIS systems that do not take advantage of Enterprise Geodatabases.  Additionally they may not have ultimate control of the authoritative parcel layers, or the CAMA vendor cannot make the tabular information available in any other form than exported Excel files.  
The following code will help you join Excel data to file geodatabase polygon layers and overwrite the data in a hosted feature layer in ArcGIS Online by uploading the file geodatabase to the portal.  

## Setup Code

```python
import arcpy, os, traceback
from arcgis.gis import GIS
from arcgis.features import FeatureLayerCollection
import zipfile
import logging

logging.basicConfig(filename = r".\Parcel_Publish_Log.txt",level=logging.INFO)
logging.info("{} Begin Moving Parcels ".format(str(datetime.datetime.now()))) 

basePath = r"C:\ProjectFolder"
shapefile = os.path.join(basePath, r"parcels.shp")

CamaExtract = os.path.join(basePath, r"ParcelsExtract.xlsx")
SalesExtract = os.path.join(basePath, r"SalesExtract.xlsx")

tempGDB = os.path.join(basePath, "Parcel_Integration.gdb")
gdbParcels = os.path.join(tempGDB, "Parcels_In")
gdbCamaExtract = os.path.join(tempGDB, "CamaExtract")
tempSales = os.path.join(tempGDB, "TaxParcelSales")

finalGDB = os.path.join(basePath, "TaxParcels.gdb")
outTaxParcels = os.path.join(finalGDB, "TaxParcels")
outSalesParcels = os.path.join(finalGDB, "TaxParcelSales")

arcpy.env.overwriteOutput = True
arcpy.env.qualifiedFieldNames = False
```

# Functions

ZIPDIR - zips the file geodatabase
FIELDLIST - generates a field mapping for appending to the final schema
GETFIELDMAP - generates an ArcPy field map for the field mappings object
GETJOINMAP - generates field map that merges the input values using a delimeter
GETSUMMAP - generates a field map that summarizes input values
INPORTOBJECTS - imports parcels and excel data to working GDB
UPDATEAGOL - overwrites the parcel service with compiled data in a zipped file geodatabase

```python
def zipdir(path, ziph):
    for root, dirs, files in os.walk(path):
        for file in files:
            if not file.endswith('.lock'):
                ziph.write(os.path.join(root, file),
                       os.path.relpath(os.path.join(root, file),
                                       os.path.join(path, '..')))

def fieldList():
    theList = []
    theList.append(("LOWPARCELID", "Parcels_In", "PPN"))
    theList.append(("PARCELID", "CamaExtract", "mpropertyNumber"))
    theList.append(("CVTTXCD", "CamaExtract", "mtaxsetCode"))
    theList.append(("SCHLTXCD", "CamaExtract", "mstateSchoolcode"))
    theList.append(("NGHBRHDCD", "CamaExtract", "NeighborhoodCode"))
    theList.append(("CLASSCD", "CamaExtract", "mClassificationID"))
    theList.append(("PRPRTYDSCRP", "CamaExtract", "mlegalDescription"))
    theList.append(("OWNERNME1", "CamaExtract", "OwnName"))
    theList.append(("PSTLCITY", "CamaExtract", "TaxpCity"))
    theList.append(("PSTLSTATE", "CamaExtract", "TaxpState"))
    theList.append(("PSTLZIP5", "CamaExtract", "TaxpZipcode"))
    theList.append(("LNDVALUE", "GovtExtract", "MKT_Land_Value"))
    theList.append(("CNTASSDVAL", "CamaExtract", "MKT_Tot_Total"))
    return theList

def salesFieldList():
    theList = []
    theList.append(("parcelid",   "mcamaid"))
    theList.append(("cvttxcd",   "ToTaxSetId"))
    #theList.append(("docname",  ""))
    theList.append(("recorddt",   "mrecordedDate"))
    theList.append(("transdt",   "mtransferDate"))
    theList.append(("grantor",   "FromDeededOwner"))
    theList.append(("grantee",   "ToDeededOwner"))
    #theList.append(("liber",  "mcamaid"))
    #theList.append(("page",  "mcamaid"))
    theList.append(("saleamnt",   "msalesAmount"))
    #theList.append(("salesratio",  "mcamaid"))
    theList.append(("resstrtype",   "Style"))
    theList.append(("resyrblt",   "YearBuilt"))
    theList.append(("resflrarea",   "FinishedLivingArea"))
    theList.append(("nghbrhdcd",   "Neighborhood"))
    theList.append(("TransferType",   "mtransferType"))
    theList.append(("UseCode",   "UseCode"))
    theList.append(("LandOnlySale",   "LandOnlySale"))
    theList.append(("SaleValid",  "msaleValid"))
    theList.append(("NumProperties",  "mnumOfProperties"))
    return theList

def getFieldMap(fms, fieldnm, inLayer, tabname, infield):
    if tabname == "":
        joinField = infield
    else:
        joinField = tabname + "." + infield
    print ("{}, {}".format(fieldnm, joinField))
    fieldmap = fms.getFieldMap(fms.findFieldMapIndex(fieldnm))
    fieldmap.addInputField(inLayer, joinField)
    fms.replaceFieldMap(fms.findFieldMapIndex(fieldnm), fieldmap)

def getJoinMap(fms, fieldnm, inLayer, tabname, fieldset):
    print(fieldset)
    fieldmap = fms.getFieldMap(fms.findFieldMapIndex(fieldnm))
    fieldmap.mergeRule = "Join"
    fieldmap.joinDelimiter = " "
    for infield in fieldset:
        if tabname == "":
            fieldmap.addInputField(inLayer, infield)
        else:
            fieldmap.addInputField(inLayer, tabname + "." + infield)
    fms.replaceFieldMap(fms.findFieldMapIndex(fieldnm), fieldmap)

def getSumMap(fms, fieldnm, inLayer, tabname, fieldset):
    print(fieldset)
    fieldmap = fms.getFieldMap(fms.findFieldMapIndex(fieldnm))
    fieldmap.mergeRule = "Sum"
    for infield in fieldset:
        if tabname == "":
            fieldmap.addInputField(inLayer, infield)
        else:
            fieldmap.addInputField(inLayer, tabname + "." + infield)
    fms.replaceFieldMap(fms.findFieldMapIndex(fieldnm), fieldmap)

def importObjects():
    try:
        print ("Importing Parcels")
        #inParcels = arcpy.CopyFeatures_management(shapefile, gdbParcels)
        
        print ("Importing Cama Extract and Sales")
        arcpy.ExcelToTable_conversion(CamaExtract, gdbCamaExtract, "Sheet1")
        arcpy.ExcelToTable_conversion(SalesExtract, tempSales, "Sheet1")

    except Exception as ex:
        print (ex)
        logging.error (str(sys.exc_info()) + "\n")
        logging.error (traceback.format_tb(sys.exc_info()[2])[0] + "..in importObjects\n")

def UpdateAGOL(zippedGDB):
    gis = GIS("https://agency.maps.arcgis.com", "username", "password")
    targetService = gis.content.get('32666ae591f04e6789f35e227388628e')
    layerColl = FeatureLayerCollection.fromitem(targetService)
    layerColl.manager.overwrite(zippedGDB)
```

##  Main Processing Block

This section joins the parcels together with the imported cama data, appends to a template geodatabase table and then uploads to ArcGIS Online. 
Keep in mind, the import of Excel data into a geodatabase can produce unpredictable results.  ArcGIS geoprocessing tools will interpret field types from data it sees in the first several rows, so it can easily misinterpret a text column for a numeric and vice versa. 

```python
try:
    
    importObjects()
    logging.info("{} Finished importing objects".format(str(datetime.datetime.now()))) 

    print ("Make Feature Layers")
    parcelview = arcpy.MakeFeatureLayer_management(gdbParcels, "inFeatures")
    camaview = arcpy.MakeTableView_management(gdbCamaExtract, "CamaExtract")

    print ("Join feature layers together")
    join1 = arcpy.AddJoin_management(parcelview, "ParcelID", camaview, "camaId")

    print ("Interrim copy of parcel data")
    copyParcels = os.path.join(tempGDB, "ParcelCopy")
    arcpy.CopyFeatures_management(join7, copyParcels)
    
    print ("Creating field mapping...")
    fms = arcpy.FieldMappings()
    fms.addTable(outTaxParcels)
    fList = fieldList()  #Get list of attributes from function
    for item in fList:
        getFieldMap(fms, item[0], copyParcels, "", item[2])
    getJoinMap(fms, "SITEADDRESS", copyParcels, "", ["mlocStrNo", "mlocStrNo2", "mlocStrDir", "mlocStrName", "mlocStrSuffix", "mlocStrSuffixDir", "msecondaryAddress"])
    getJoinMap(fms, "PSTLADDRESS", copyParcels, "", ["mlocStrNo", "mlocStrNo2", "mlocStrDir", "mlocStrName", "mlocStrSuffix", "mlocStrSuffixDir", "msecondaryAddress"])
    getSumMap(fms, "TOTCNTTXOD", copyParcels, "", ["priorDelqOwedTot", "firstHalfTotOwed", "secondHalfTotOwed"])

    print ("Deleting Production Features...")
    arcpy.TruncateTable_management(outTaxParcels)

    print ("Appending Production Features...")
    arcpy.Append_management(copyParcels, outTaxParcels, "NO_TEST", fms)

    #CREATE SALES LAYER - This is a one to many relationship
    print ("Creating Sales Data")
    parcelView2 = arcpy.MakeFeatureLayer_management(gdbParcels, "inFeatures2")
    salesview = arcpy.MakeTableView_management(tempSales, "Sales")
    salesJoin = arcpy.AddJoin_management(parcelView2, "ParcelID", salesview, "camaId", "KEEP_COMMON")

    print ("Interrim copy of joined data")
    copySales = os.path.join(tempGDB, "SalesCopy")
    arcpy.CopyFeatures_management(salesJoin, copySales)

    print ("Creating field mapping...")
    fms2 = arcpy.FieldMappings()
    fms2.addTable(tempSales)
    fList = salesFieldList()
    for item in fList:
        getFieldMap(fms2, item[0], copySales, "", item[1])
    getJoinMap(fms2, "fulladdr", copySales, "", [ "mlocStrNo",  "mlocStrDir",  "mlocStrName",  "mlocStrSuffix",  "mlocStrSuffixDir",  "msecondaryAddress"])
    getSumMap(fms2, "cntassdval", copySales, "", [ "mlandValue",  "mimprovementValue"])

    print ("Updating Sales Polygons...")
    arcpy.TruncateTable_management(tempSales)
    arcpy.Append_management(copySales, tempSales, "NO_TEST", fms2)
    
    print ("Updating Sales Points...")
    arcpy.FeatureToPoint_management(tempSales, outSalesParcels, "INSIDE")

    print ("Zipping GDB...")
    zippedGDB = os.path.join(basePath, "ParcelCAMA.gdb.zip")
    zipObj = zipfile.ZipFile(zippedGDB, 'w', zipfile.ZIP_DEFLATED)
    zipdir(finalGDB, zipObj)
    zipObj.close()

    #Upload to ArcGIS Online
    print ("Uploading Parcels")
    logging.info ("{} Begin Uploading Parcels ".format(str(datetime.datetime.now())))
    UpdateAGOL(zippedGDB)

except Exception as ex:
    print (ex)
    logging.error (str(sys.exc_info()) + "\n")
    logging.error (traceback.format_tb(sys.exc_info()[2])[0] + "\n")

print ("Finished!")
logging.info("{} Finished! ".format(str(datetime.datetime.now())))
```
