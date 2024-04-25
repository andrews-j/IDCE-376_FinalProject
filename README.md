# Urban Greenery and Demographics in Worcester, MA

### IDCE 376, Spatial Database Management, Spring 2024
#### Jason Andrews, Clark University MSGIS, 2024

This repository contains submissions pertaining to final project work for IDCE 376.

The assignment description can be viewed in the *Final_Project-Rubric.pdf* document.

# Introduction

### Objective: 
Combine infomation from census and ASC data to create an HDI by census tract layer for Worcester, and compare this to metrics of urban canopy/greenness.

### Research Questions:
- How does urban canopy correlate with our combined measure of economic mobility, educational attainment, and population health?
- Has this relationship changed over time?

### Study Area:

The study area is the city limits of Worcester, and the spatial scale of the analysis will be the 44 census tracts in Worcester, as visualized below:

![Screenshot 2024-04-14 at 9 27 33 PM](https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/5a63beb5-057f-4372-af2c-cbc16d154f6a)

## Part 1: Data Aquisition

For  information on where data was obtained, please see DataSources.md

### Vector Layers:

**2010 and 2020 Census**

These layers contain information about total population, and racial composition for each census tract. 

See **Cenus2010_cleaning.ipynb** and **Cenus2020_cleaning.ipynb** in the Scripts folder.

This is a subset of the 2010 census data:

![Screenshot 2024-04-14 at 9 30 35 PM](https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/7008a268-850c-4f0c-860a-d656f19596c2)

**CDC Life Expectancy by Census Tract**

See **CDC_LE_Processing.ipynb**

<img width="988" alt="Screenshot 2024-04-23 at 5 24 03 PM" src="https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/518b066e-ffc0-4c58-8fdc-acd15e518d1d">

**American Community Survey-- Percent of Population with Income below Federal Poverty Line**

Both ASC layers were manually cleaned in QGIS, with further cleaning in python. The most tedious part of this entire process was merging census areas that were split betwen the 2010 and 2020 census.

See **ACS_2020_Merge.ipynb**

<img width="907" alt="Screenshot 2024-04-23 at 5 28 13 PM" src="https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/fec9c855-032f-4ec7-b7f4-874d48ad379d">

**American Community Survey-- Percent of Population with Bachelor's Degrees**

<img width="885" alt="Screenshot 2024-04-23 at 5 27 22 PM" src="https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/44e13066-00a2-4863-8861-f135bee7ee15">

**Tree Planting Points**

Point layer of nearly 10,000 trees planted in Worcester County in 2010-2012 through a Massachussets Department of Conservation and Recreation (MA DCR) tree planting program.

![Screenshot 2024-04-14 at 9 35 29 PM](https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/f1710f8f-d12c-4d8e-834c-b92846e08c8e)

**Canopy Cover Maps**

Worcester canopy cover vector layers from 2008, 2010, and 2015.

Canopy cover 2010 detail:

<img width="609" alt="Screenshot 2024-04-15 at 1 38 04 PM" src="https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/a1878029-529c-48d9-a247-01cd1bda83af">


### Raster Layers:

Raster images include NDBI (Normalized Differenced Built Index), NDVI (Normalized Differenced Vegetation Index), and UVI (Urban Vegetation Index) from 5 different time points: 2007, 2011, 2015, 2019, and 2023, clipped to Worcester city limits. 

Raster images were aquired with PySTAC. Each is a median composite image between March 15 and September 15 of the chosen year. See GetWorcesterImages.ipynb.

All data is kept in, or reprojected to EPSG 32619 WGS 84/ UTM Zone 19N.

Examples:

**NDBI 2007**

![Screenshot 2024-04-14 at 9 42 27 PM](https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/45c41895-83cd-4436-9c0f-aa02297040d5)

**NDVI 2015**

![Screenshot 2024-04-14 at 9 44 33 PM](https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/c88b0a3a-00e9-4f8e-8cdc-a72ef3384706)

**UVI 2023**

![Screenshot 2024-04-14 at 9 45 19 PM](https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/9b0eef3e-7da8-4072-a880-4ab88805d2b3)

## Part 2: Importing Data to SQL

**See *ShellCommands.sh* for more details**

This is the script I used in CLI to batch import vector files into my database, which is called finalProj:

```bash

for shapefile in ./*.shp; do
    
    filename=$(basename "$shapefile" .shp)
    
    shp2pgsql -s 32619 -I "$shapefile" "$schema_name.$filename" >> vectorImport.sql
done

```


This is the script I used in CLI to batch import raster files into my database, which is called finalProj:

```bash

for raster in ./*.tif; do
    
    filename=$(basename "$raster" .tif)
    
    raster2pgsql -s 32619 -I -C -M "$raster" "$schema_name.$filename" >> rasterImport.sql
done

```

Once the rasters are added to the database they must be converted to vector. This is the SQL batch command used to convert all ndbi, ndvi, and uvi images in the database to points.
See finalProj_SQL.sql for more details.

```sql
DO $$
DECLARE
    raster_table_name TEXT;
    vector_table_name TEXT;
BEGIN
    FOR raster_table_name IN SELECT table_name FROM information_schema.tables WHERE table_name LIKE 'ndvi_%' OR table_name LIKE 'ndbi_%' OR table_name LIKE 'uvi_%' LOOP
        vector_table_name := raster_table_name || '_points';
        EXECUTE FORMAT('CREATE TABLE %I AS 
                        SELECT CAST((ST_PixelAsPoints(rast)).val AS DECIMAL) AS float_val, 
                               (ST_PixelAsPoints(rast)).*
                        FROM %I;', vector_table_name, raster_table_name);
    END LOOP;
END $$;
```

The result, at this point, is that we have quite a lot of tables, 41 to be exact.

<img width="354" alt="Screenshot 2024-04-23 at 6 09 33 PM" src="https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/5f62c549-f437-4ccc-99a9-492039556156">

## Part 3: Create Composite HDI

Now we are going to use the life expectancy estimate, percent of population in poverty, and percent of population with a bachelor's degree to create an HDI measure for each tract. To do this we will normalize each measure to an index between 0 and 1, which will then be used to weight each evenly in our final composite index.

See FinalProj_SQL.sql for more details, but here is the fun part, combining the relevant columns to a single table:

```SQL
CREATE TABLE hdi_calc AS
SELECT
    l.tract,
    l."life expec",
    p.povper,
    e.perbach
FROM
    le_tracts l
JOIN
    woo_poverty_2020 p ON l.tract = p.tract
JOIN
    woo_education_2020 e ON l.tract = e.tract;
```
This is what our table looks like:

<img width="1017" alt="Screenshot 2024-04-23 at 7 21 02 PM" src="https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/06fc47c8-6b8f-4a87-9338-e0e91ff07fb7">


Now we will calculate a 0-1 normalized value for each
```SQL
SELECT 
    MIN(life_exp) AS min_life_exp,
    MAX(life_exp) AS max_life_exp,
    MIN(povper) AS min_povper,
    MAX(povper) AS max_povper,
    MIN(perbach) AS min_perbach,
    MAX(perbach) AS max_perbach
INTO 
    min_max_values
FROM 
    hdi_calc;

ALTER TABLE hdi_calc
ADD COLUMN pov_norm NUMERIC,
ADD COLUMN ed_norm NUMERIC,
ADD COLUMN le_norm NUMERIC;

UPDATE hdi_calc
SET 
    le_norm = (life_exp - (SELECT min_life_exp FROM min_max_values)) / 
                          ((SELECT max_life_exp FROM min_max_values) - (SELECT min_life_exp FROM min_max_values)),
    pov_norm = (povper - (SELECT min_povper FROM min_max_values)) / 
                        ((SELECT max_povper FROM min_max_values) - (SELECT min_povper FROM min_max_values)),
    ed_norm = (perbach - (SELECT min_perbach FROM min_max_values)) / 
                         ((SELECT max_perbach FROM min_max_values) - (SELECT min_perbach FROM min_max_values));
```
Which brings us to here:

<img width="1033" alt="Screenshot 2024-04-24 at 8 46 46 PM" src="https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/9b0b7359-a081-4633-b0a3-4777d4526080">

Notice that there are a few tracts without life expectancy data. We will subsitute an average value in those cases.

![Screenshot 2024-04-24 at 10 12 02 PM](https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/c2cddbef-93c8-412b-8504-16057ff5edb4)

![Screenshot 2024-04-24 at 10 11 18 PM](https://github.com/andrews-j/IDCE-376_FinalProject/assets/26927475/00052bb6-a787-4ca7-a61f-96ae9ed1eb76)

