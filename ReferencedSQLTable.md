## Generate registered read-only table using SQL

You can generate a table result of joining the parcels with the CAMA table using a SQL Stored Procedure.  The resulting table can be registered with the geodatabase, and published as a feature service.  
It is recommended that you NOT register the table as archived or versioned, as managing the table using SQL will not be possible.  
The table will contain a generated ObjectID and the shape field from the parcels will be brought over.  

Assumes the CAMA table is in the same database as the parcel layer. 
Assumes the output table has been created already (you can use the same SQL code, but just the main block with an "into schema.tablename" phrase after the select statement to construct the table first time).

The resulting registered table (as a polygon feature class) can be published to ArcGIS Portal as a referenced dataset.  The stored procedure containing this data can be scheduled to truncate and append the data from the table, which would then be reflected in the referenced service in ArcGIS Enterprise.  
If the layer needs to be used in ArcGIS Online, one option here could be an item referencing the data in the local Enterprise with embedded credentials.  Collaborating the layer using ArcGIS Distributed Collaboration would not be possible because collaboration as a copy cannot be used unless the data was registered as versioned or archived. 

You can also make this code a view instead of a stored procedure updating a static table.  However, parcel data can be large, and the performance may not be ideal.  

```sql
truncate table dataowner.CC_Parcel_CAMA

insert into dataowner.Parcel_CAMA (ObjectID, PARCELID,ASSESSORNUM
	, BUILDING, UNIT, STATEDAREA, CVTTXCD, CVTTXDSCRP, SCHLDSCRP, SCHLTXCD, USEDC
	, NGHBRHDCD, CLASSCD, CLASSDSCRP, SITEADDRESS, SITECITY, SITEZIP, MUNICIPALITY
	, PRPRTYDSCRP, CNVYNAME, OWNERNME1, OWNERNME2, RESYRBLT, LNDVALUE, BLDGVALUE, TOTALVALUE
	, RECORDDT, TRANSDT, GRANTOR, GRANTEE, SALEAMNT, 
	Shape, GDB_GEOMATTR_DATA)
select ObjectID = cast(row_number() over(order by b.ParcelID) as int),
	b.PARCELID,b.ASSESSORNUM
,b.BUILDING,b.UNIT,b.STATEDAREA,b.CVTTXCD,b.CVTTXDSCRP,b.SCHLDSCRP,b.SCHLTXCD,b.USEDC
,b.NGHBRHDCD,b.CLASSCD,b.CLASSDSCRP,b.SITEADDRESS,b.SITECITY,b.SITEZIP,b.MUNICIPALITY
,b.PRPRTYDSCRP,b.CNVYNAME,b.OWNERNME1,b.OWNERNME2,b.RESYRBLT,b.LNDVALUE,b.BLDGVALUE,b.TOTALVALUE
,b.RECORDDT,b.TRANSDT,b.GRANTOR,b.GRANTEE,b.SALEAMNT, 
a.shape, NULL
--into #temp
from dataowner.TaxParcels a
	left outer join dataowner.CAMATable b on a.ASSESSOR_NUM=b.assessornum
```

Note that in this code we are building the ObjectID on the fly.  Just in case there are one-to-many relationships here, some parcels may be represented multiple times, so instead of using the parcel ObjectID outright, a new one is built so that each row has a unique value.  As long as the output registered feature class table isn't edited or managed by ArcGIS (reading/querying only) then this workflow is viable. 
