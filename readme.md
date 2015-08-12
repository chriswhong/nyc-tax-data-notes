#NYC Tax Data Notes

In collaboration with @talos, we are assembling the most comprehensive property tax dataset for New York City from publicly available information sources.

The city's property tax bills contain a tremendous amount of data, including how taxes are calculated for each lot and condominium unit, and what extemptions and abatements exist for each.

This repo documents the data cleansing and transformation process.

##Data Source

This project builds on @talos' [NYC Stabilization Unit Counts](https://github.com/talos/nyc-stabilization-unit-counts), which sought a subset of exemptions included in a subset of NYC properties in order to get a solid count of rent-stabilized units.  To achieve this, he downloaded and archived hundreds of thousands of current and historic tax bills from the New York City Department of Finance (DoF) website. These are available in a borough/block/lot directory structure at [taxbills.nyc](taxbills.nyc).  The bills were then scraped using a python script to extract the raw data.  Read more about the process in [the repo's readme](https://github.com/talos/nyc-stabilization-unit-counts).

The rent stabilization project only gathered data for buildings with 6 or more residential units, as those are the only ones subject to rent stabilization laws.

For this project, the intent is to gather *100% of the tax bill data for the entire city for a single year's tax levy*.  The latest bills were sent on June 5th, 2015.  Using the same methodology as in the rent stabilization project we downloaded every bill for June 5th, including those for condominium units, over 1.1 million bills in total.

The output of the script that pulls data from the pdfs is a file called `rawdata.csv`, which is avaiable at [taxbills.nyc](http://www.taxbills.nyc)
This contains raw data in a key/value schema, so one bill can produce many rows in this csv, and it includes data from all bills currently archived at taxbills.nyc.

##Data Cleaning

1) Import `rawdata.csv` into postgresql

```
CREATE TABLE rawdata
(
  bbl bigint,
  activitythrough date,
  section text,
  key text,
  duedate date,
  activitydate date,
  avalue text,
  meta text,
  apts text
)
```
import using pgadmin or whatever other means you prefer

2) Extract top-level data from the June 5th tax levy, and pivot it so that the result set includes one row per BBL.  This table will not include exemptions and abatements.

```
SELECT bbl,
     MAX(CASE WHEN key = 'Owner name' THEN avalue END ) AS ownername,
     MAX(CASE WHEN key = 'Mailing address' THEN avalue END ) AS address,
     MAX(CASE WHEN key = 'tax class' THEN avalue END ) AS taxclass,
     MAX(CASE WHEN key = 'current tax rate' THEN avalue END ) AS taxrate,
     MAX(CASE WHEN key = 'estimated market value' THEN avalue END )::double precision AS emv,
     MAX(CASE WHEN key = 'tax before exemptions and abatements' THEN avalue END )::double precision AS tbea,
     MAX(CASE WHEN key = 'tax before abatements' THEN avalue END )::double precision AS tba,
     MAX(CASE WHEN key = 'annual property tax' THEN avalue END )::double precision AS propertytax
  FROM rawdata
  WHERE activitythrough = '2015-06-05'
 GROUP BY bbl
 ORDER BY bbl
```
Load it into a new table, `june15all`

```
CREATE TABLE june15all
(
  bbl bigint,
  ownername text,
  address text,
  taxclass text,
  taxrate text,
  emv double precision,
  tbea double precision,
  tba double precision,
  propertytax double precision
)
```
## Some Queries

Only Condominium Units

```
SELECT * FROM june15all
WHERE bbl % 10000 > 1000
AND bbl % 10000 < 7501
```
Only Non-Condominiums
```
SELECT * FROM june5all 
WHERE bbl % 10000 < 1001
OR bbl % 10000 > 7500
```
## Todo
- Pull out exemptions and abatements from `rawdata` into their own table

- Add Lot BBLs (parent BBLs) to all condo BBLs using the assessment roll as a lookup table.  This will allow the summation of condo unit data into a single number for the physical lot.