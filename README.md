# CAMA_Integration
SQL and Python scripting methods to produce parcels with CAMA data for forward facing applications

## Introduction

[Computer Assisted Mass Appraisal](https://en.wikipedia.org/wiki/Real_estate_appraisal#:~:text=Computer%2Dassisted%20mass%20appraisal) (CAMA) data is used by most county tax management agencies around the United States.  Depending on the size of the county, these systems can reside internally or externally and are generally managed by software developed by a 3rd party.  In many cases this application is Saas and the data resides on the software vendor's servers out of the control of the county.  Esri works with Counties to develop applications in ArcGIS Online and ArcGIS Enterprise that use this data together with parcel polygon spatial data to display and analyze attributes from the CAMA systems in a spatial context.

There are many ways to do this, and there are considerations that need to be made when determining the best method for the agency.  These include the location of the authoritative parcel data, the source of the CAMA data, coordination between the county and the vendor in developing methods to get data in a form that can be merged with the parcels.  Considerations also need to be made regarding what information is needed. For instance is the data going to represent the current state of the parcel, include historic parcels, represent multiple owners (for instance in the case of a single parcel that houses condominium owners) or sales data.  

##  Methods

There are many methods, most of them involving some sort of custom automation.   Each one somewhat depends on the system and how available the CAMA data is.  In the case of Python, there are a few different ways to handle it. I'll illustrate three common methods in the pages on this repository.

  1. [Python File Geodatabase transfer](FileGDB_Overwrite.md)
  2. [Python truncate and append](Append_FS.md)
  3. [SQL view published to Enterprise](ReferencedSQLTable.md)

## CAMA data preparation

CAMA data tends to be stored as multiple tables in complex one to many relationships.  One parcel id related to tables of multiple permit requests, etc.  In order to use this data joined to parcel polygons decisions need to be made regarding how to flatten the data so that it can be joined to the single parcel records for use in web maps.  Does the data need to be flattened so there is one record per parcel.  If multiple records need to be considered, is the output the same polygon represented a number of times for each related record from the CAMA data.  This can occur if the parcel was sold multiple times over the years and the end application needs to reference each sale individually. 

The flattened CAMA table, which could just be a database view, needs to be accessible by the process joining the parcels, whether by SQL or by Python.  

