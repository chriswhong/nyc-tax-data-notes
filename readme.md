#NYC Tax Data Notes

We are assembling a comprehensive property tax dataset for New York City from publicly available information sources.

The city's property tax bills contain a tremendous amount of data, including how taxes are calculated for each lot and condominium unit, and what extemptions and abatements exist for each.

This repo documents the data cleansing and transformation process.

##Data Source

This project builds on @talos' [NYC Stabilization Unit Counts](https://github.com/talos/nyc-stabilization-unit-counts), which sought a subset of exemptions included in a subset of NYC properties in order to get a solid count of rent-stabilized units.  To achieve this, he downloaded and archived hundreds of thousands of current and historic tax bills from the New York City Department of Finance (DoF) website. These are available in a borough/block/lot directory structure at [taxbills.nyc](taxbills.nyc).  The bills were then scraped using a python script to extract the raw data.  Read more about the process in [the repo's readme](https://github.com/talos/nyc-stabilization-unit-counts).

The rent stabilization project only gathered data for buildings with 6 or more residential units, as those are the only ones subject to rent stabilization laws.

For this project, the intent is to gather *100% of the tax bill data for the entire city for a single year's tax levy*.  The latest bills were sent on June 5th, 2015.  Using the same methodology as in the rent stabilization project we downloaded every bill for June 5th, including those for condominium units, over 1.1 million bills in total.

The output of the script that pulls data from the pdfs is a file called `rawdata.csv`, which is avaiable at [taxbills.nyc](http://www.taxbills.nyc)
This contains raw data in a key/value schema, so one bill can produce many rows in this csv, and it includes data from all bills currently archived at taxbills.nyc.

##Data Cleaning

For this project, we'll create two tables from `rawdata.csv`.  The first, `june15bbls` will include the "top-level" data for each BBL (total tax, estimated market value, billable assessed value, etc).  `june15exab' will include the exemptions and abatements, which have a many-to-one relationship with BBLs.

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
import using pgadmin>import or whatever method works for you


2) Create `june15bbls` by pivoting the top-level data into one row per BBL.  This step includes a join with table called `condolookup`, which contains the condominium number for all condo unit and lot bbls. The addition of the condonumber will allow for aggregation of taxes paid by the units of a single condo building. (this lookup table was derived from RPAD)

```
CREATE TABLE june15bbls as
  SELECT a.bbl,
     MAX(CASE WHEN key = 'Owner name' THEN avalue END ) AS ownername,
     MAX(CASE WHEN key = 'Mailing address' THEN avalue END ) AS address,
     MAX(CASE WHEN key = 'tax class' THEN avalue END ) AS taxclass,
     MAX(CASE WHEN key = 'current tax rate' THEN avalue END ) AS taxrate,
     MAX(CASE WHEN key = 'estimated market value' THEN avalue END )::double precision AS emv,
     MAX(CASE WHEN key = 'tax before exemptions and abatements' THEN avalue END )::double precision AS tbea,
     --billable assessed value ended up in the meta column of tbea
     MAX(CASE WHEN key = 'tax before exemptions and abatements' THEN meta END )::double precision AS bav,
     MAX(CASE WHEN key = 'tax before abatements' THEN avalue END )::double precision AS tba,
     MAX(CASE WHEN key = 'annual property tax' THEN avalue END )::double precision AS propertytax,
     --join with condolookup to include condonum if the bbl is for a condo unit or lot
     MAX(b.condono) as condonumber,
     --make a condolot field for easy querying
     CASE 
      WHEN a.bbl % 10000 > 7500 AND a.bbl % 10000 < 7600 THEN 'lot' 
      WHEN a.bbl % 10000 > 1000 AND a.bbl % 10000 < 7000 THEN 'unit'
      ELSE null END AS condolot

  FROM rawdata a
  LEFT JOIN condolookup b ON a.bbl = b.bbl
  WHERE activitythrough = '2015-06-05'
  GROUP BY a.bbl
  ORDER BY a.bbl
```

3) Create `june15exab` table that includes one row per exemption/abatement 

```
CREATE TABLE june15exab AS
SELECT 
bbl,
CASE 
 WHEN section = 'details-abatements' THEN 'abatement'
 WHEN section = 'details-exemptions' THEN 'exemption'
END AS type,
key as detail,
avalue as amount
FROM rawdata 
WHERE activitythrough = '2015-06-05'
AND (
    section = 'details-exemptions' OR section = 'details-abatements' 
)
```

## A Few Good Queries 

Count condos vs non-condos
```
SELECT condo,count(*) FROM june15bbls 
GROUP BY condo
```
"false";847959
"true";233665

Sum all annual tax by borough 
```
SELECT LEFT(bbl::text,1)as borough, SUM(propertytax) as totaltax
FROM june15bbls 
GROUP BY borough
```

Sum tax before exemptions and abatements, sum annual tax
```
SELECT SUM(tbea),SUM(propertytax) FROM june15bbls
```
