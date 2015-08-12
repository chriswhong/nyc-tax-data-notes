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

To 