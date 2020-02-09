# almost-everywhere

## Data prep

### Reprojection

The 311 data should be [reprojected](https://www.nceas.ucsb.edu/scicomp/recipes/gdal-reproject) to EPSG:2263 to match Planimetrics (you could also do the reverse but Planimetrics is multiple tables and a more complicated format).

You can do this ahead of time:

```bash
ogr2ogr -t_srs EPSG:2263 311_Service_Requests_from_2010_to_Present_2263.csv 311_Service_Requests_from_2010_to_Present.csv
```

Or wait to do it with PostGIS using `ST_Transform`.

### Cut away most 311 columns

For spatial analysis we really only need three columns: `unique_key`, `latitude`, and `longitude` (a.k.a. columns 1, 39 and 40):

```bash
csvcut -c 1,39,40 311_Service_Requests_from_2010_to_Present_2263.csv > slim311.csv
```


## PostgreSQL + PostGIS

### Installation

Follow [PostgreSQL and PostGIS Installation in MacOS](https://medium.com/@Umesh_Kafle/postgresql-and-postgis-installation-in-mac-os-87fa98a6814d)

### Import data



#### 311

Create a table with the proper schema:

```sql
CREATE TABLE service_requests;

ALTER TABLE service_requests ADD COLUMN unique_key VARCHAR(8);
ALTER TABLE service_requests ADD COLUMN latitude NUMERIC;
ALTER TABLE service_requests ADD COLUMN longitude NUMERIC;
```

Copy in the cut csv:

```sql
COPY service_requests FROM '/Users/abe.rubenstein/Downloads/slim311.csv' DELIMITER ',' CSV HEADER;
```

Add new geometry column [[source](https://medium.com/@paylakatel/part-2-postgis-at-the-city-of-boston-711cf30cf1f3)]:

```sql
SELECT AddGeometryColumn ('public', 'service_requests', 'the_geom', 4326, 'POINT', 2);

UPDATE service_requests SET the_geom = ST_SetSRID(
  ST_MakePoint(
    "longitude"::double precision,
    "latitude"::double precision
  ),
  4326 # this SRID depends on whether or not you initially reprojected 
);
```

**If you still need to reproject, add a new column to hold EPSG:2263 geometry and fill it.**

```sql
SELECT AddGeometryColumn ('public', 'service_requests','the_geom_2263',2263,'POINT',2);

UPDATE service_requests SET the_geom_2263 = ST_Transform(the_geom, 2263);
```

#### Planimetrics

Copy the FileGDB into Postgres [[source](https://gis.stackexchange.com/a/83325/52312)]. This will create several tables within the specified db.

```bash
ogr2ogr -f PostgreSQL PG:"host=localhost port=5432 dbname=<YOUR_DB_NAME> user=<YOUR_USER>" NYC_DoITT_Planimetric_OpenData.gdb -overwrite -progress -skipfailures --config PG_USE_COPY YES
```

> `-skipfailures` is necessary because some shapes in the FileGDB seem to be misclassified. This happens in QGIS as well.

### Analysis

Best query so far (takes 20-30min):

```sql
SELECT count(*) FROM roadbed;
```
```
 count 
-------
 92389
(1 row)
```

```sql
SELECT COUNT(DISTINCT(objectid)) FROM (SELECT * FROM roadbed) AS s JOIN service_requests AS p ON ST_Contains(s.shape, p.the_geom_2263);
```

```
 count 
-------
 77537
(1 row)
```

I * think * this means that 84% of `roadbed` features contain at least one point from 311.
