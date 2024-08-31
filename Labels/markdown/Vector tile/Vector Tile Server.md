## Vector Tile Server
As the name implies, a vector tile server is a server that stores and provides vector tiles. The tile is requested using a specific URL with the tile coordinates as URL parameters. Furthermore, if properly configured, it is also possible to define the type of vector tile it should return (e.g. only provide peaks, or provide all features merged into one tile).

### Alternatives

There are a couple of options we considered before finally choosing [Martin](https://martin.maplibre.org/) as our vector tile server framework:

#### Basemap
The first option we considered was using [Basemap](https://basemap.at/standard-5/) (Standard 5) as our vector tile server. The server is externally hosted and contains many features in Austria that are relevant for our application. The data is also free to be used and is updated every 2 months. The main issue we encountered with Basemap were inconsistent data behaviors.

One such example are duplicate data entries with slightly differing metadata (e.g. two different altitudes and different positions of the same peak). We are assuming that this data error stems from the merging of two (or more) data sources.

While we tried to minimize this behavior by using custom preprocessing steps (e.g. if multiple mountains with same/similar names are around a certain radius only show the one with the highest altitude), we found that this approach was not ideal.

Another possible problem we encountered was that the vector tiles provided by the Basemap contained a lot of information that is currently not used in our application. Which would cause a greater overhead for our parser and cause more data being stored on the cache of the user (the complete/unparsed vector tile is being cached).

The last concern we had with Basemap, was that this is an external service. Every outage, problems with data sources and problems with scalability due to increased traffic demands were out of our hands. We therefore concluded that having our own vector tile server was the ideal solution for our application.

#### Planetiler
[Planetiler](https://github.com/onthegomap/planetiler) is an application that can be used to quickly and easily extract features from an OpenStreetMap data source. Using Planetiler allowed us to specify which features and what meta information (location, altitude, name, …) we wanted to retain. The application further allowed us to export all the features into a [PMTiles archive](https://github.com/protomaps/PMTiles). 

A PMTiles archive is a single file which stores all the vector tile data in a binary form. The [PMTiles repository](https://github.com/protomaps/PMTiles) provides an application that can be used to act as a tile server that provides individual vector tiles from this archive file.

In our first iteration of our own vector tile server we used Planetiler and PMTiles. This solution worked good enough for prototypes purposes. Unfortunately we quickly found that the maximum zoom level of PMTiles was restricted to a zoom level of 15. Any zoom level beyond this point could not be stored/retrieved from the PMTiles archive. This limitation resulted in a problem on the AlpineMaps application since the tile requests were tightly connected to other tile types like the texture of the ground and the height of the terrain. This connection couldn't be solved easily at this point in time. We therefore had to find another solution to this problem.
### Martin
The [Martin](https://martin.maplibre.org/) tile server is the current vector tile server infrastructure we are using. The server can be easily installed and uses a [PostgreSQL](https://www.postgresql.org/)(Postgres for short) database with the [PostGIS](https://postgis.net/) extension for data storage and the possibility for data preprocessing. The data stored in the SQL database is coming from OpenStreetMap using an importer such as [osm2pgsql](https://osm2pgsql.org). While Martin can also serve a simple PMTiles archive, the main ways to serve data from the SQL database are by using SQL Views or SQL Functions.

#### Views
The most simple way to serve vector tiles from Martin is to create a view of the data you want to serve and specify this view in your config.yaml file. Please note that Martin requires one field for coordinates so that the vector tiles can be calculated (in the below example planet\_osm\_point.way).

**Example:**
peak view:
``` sql
CREATE VIEW peaks AS 
	SELECT planet_osm_point.name,
	    planet_osm_point.tags->'ele'::text AS ele,
	    planet_osm_point.way -- necessary field for coordinates
   FROM planet_osm_point
  WHERE planet_osm_point."natural" = 'peak'::text;
```

peak config:
```yaml
postgres:
  ...
  tables:
    peaks:
      schema: public
      table: peaks
      srid: 3857
      geometry_column: way
      geometry_type: POINT
      properties:
        name: text
        ele: int
...
```

Although views can be used to provide the data very fast with limited configurations needed, it also doesn't give you too many options in fine-tuning how much and what is served at which zoom levels.
#### Functions
The alternative to views are functions as the tile source (https://maplibre.org/martin/sources-pg-functions.html)

In PostgreSQL you can specify functions that are to be executed. In those functions more freedom is given in how detailed you want to configure your query. Martin requires the function to have 3 integer parameters (z,x,y values for the tile you are currently requesting) and a bytea as a return type. The bytea is written by the ST\_AsMVT function that is available in PostGIS. Additionally it is also possible to specify a fourth parameter where you can pass through additional values to your function from the requesting URL.

**Example:**
peak function:
```sql
CREATE OR REPLACE
    FUNCTION peak_tile(z integer, x integer, y integer)
    RETURNS bytea AS $$
DECLARE
    mvt bytea;
BEGIN
    -- NOTE it is possible to add "if branches" for better individual configurations
    SELECT INTO mvt ST_AsMVT(tile, 'peak_tile', 4096, 'geom', 'id') FROM (
        SELECT
            id, name, long, lat,
            ele,
           
            -- transform the point into the current vector tile -> x/y points lie within [0,4096] pixels
            -- this is necessary for vector tiles on a flat map like leaflet
            ST_AsMVTGeom(
                ST_Transform(way, 3857),
                ST_TileEnvelope(z,x,y),
                4096, 64, true
            ) as geom

        FROM peaks
        -- only select points where the current position and current tile bbox overlap
        WHERE ST_TRANSFORM(way,4674) && ST_Transform(ST_TileEnvelope(z,x,y), 4674)
        ORDER BY ele DESC
        -- possible to use z parameter here to to differentiate how much is returned per zoom level
        LIMIT 1
    ) as tile;

  RETURN mvt;
END
$$ LANGUAGE plpgsql IMMUTABLE STRICT PARALLEL SAFE;
```

peak config:
```yaml
postgres:
  ...
  functions:
    peak_tile:
      schema: public
      function: peak_tile
      minzoom: 7
      properties:
        name: text
        long: double
        lat: double
        ele: int
...
```
