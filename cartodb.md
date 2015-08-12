#Some CartoDB Queries

An export of only physical tax lots (non-condo lots) was imported into cartoDB, to be joined with geometries from MapPLUTO.

Get only the physical tax lots from `june15all`

```
SELECT * FROM june15all 
WHERE bbl % 10000 < 1001
OR bbl % 10000 > 7500
```

Create a boolean condo column in the result set, based on the lot (identify condo  lots in a table that contains ALL physical lots):
```
SELECT a.cartodb_id, a.ownername,a.address,a.the_geom,a.the_geom_webmercator,a.bbl, b.propertytax, CASE WHEN (a.bbl::bigint % 10000 > 7500 AND a.bbl::bigint % 10000 < 7600) THEN 'true' ELSE 'false' END as condo
FROM pluto15v1 a
LEFT JOIN june15lots b
ON a.bbl = b.bbl
```

Join MapPLUTO with tax lot data

```
SELECT a.cartodb_id, a.ownername,a.address,a.the_geom,a.the_geom_webmercator,a.bbl, b.propertytax
FROM pluto15v1 a
LEFT JOIN june15lots b
ON a.bbl = b.bbl
```

