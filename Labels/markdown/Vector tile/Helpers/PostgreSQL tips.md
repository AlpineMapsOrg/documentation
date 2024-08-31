# PostgreSQL tips
## SQL commands from files
For ease of development we are currently using .sql files where we are writing our queries, which will then be send to the postgres server and exectuted as a batch command:
``` sh
psql alpinemaps -U alpine -f query.sql
```


In order to avoid having to type in your password at every terminal command you can define the environment variable PGPASSWORD and set it to your password:
``` sh
export PGPASSWORD=SOMEPASSWORD
```



## Postgres commands
```
# useable once you are within the postgres shell (e.g. by using the psql command)
# show all tables
\dt

# show all views
\dv

# show structure and definition of table/view named peaks
\d+ peaks

# quit
\q
```
## Various PostGIS and Mapbox Functions
https://postgis.net/docs/index.html
``` sql
-- extract x/y coordinate
ST_X
ST_Y
-- extract a coordinate field as a text
ST_AsText
-- transform from one coordinate system to another
ST_TRANSFORM
-- transforms coordinates to mapbox tile (all coordinates relative to left upper origin point of tile bounding box)
ST_AsMVTGeom
-- creates a tile bounding box of the following z,x,y values
ST_TileEnvelope

example: 
ST_X(ST_TRANSFORM(way,4674)) AS LONG,
	ST_Y(ST_TRANSFORM(way,4674)) AS LAT,
	name,
	ST_AsMVTGeom(
		ST_Transform(way, 3857),
		ST_TileEnvelope(13,4384,2878),
		4096, 64, true
	) as geom,
	ST_AsText(ST_Transform(ST_TileEnvelope(13,4384,2878), 4326))
```
## Coordinate system 
EPSG:3857:
uses meters as units (although inaccuracies may occur nearer to the poles)
## Indices
https://www.postgresql.org/docs/current/indexes-types.html