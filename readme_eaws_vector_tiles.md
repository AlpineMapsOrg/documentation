Author: Jörg Christian Reiher, Last Update: 24.04. 2024

# EAWS Regions
EAWS Regions are the geographic regions for which avalanche forecasts are issued by the European Avalanche Warning Services (EAWS)[1]. Every region has a name (e.g. AT-02) and a polygon describing its boundary. Regions can change over time. This is why regions also have a start date and an end date describing the timespan within which this regions was valid.  The website avalanche.org provides GeoJson files for all regions (including outdated) to endure that older bulletins can be read. The folder “latest” only contains the currently valid regions though. The latest regions have an end date that is not defined (e.g. null). The vertices of each region’s boundary polygon are obtained from mapbox vector tile files. These vector tile files must be retrieved from a tileset which is served by a server. How this is done is explained in the following sections.

# Vector Tiles
Map data is stored in tiled format. That means depending on the zoom level, the world is divided into a certain amount of tiles. Zoom level 0 has only one tile covering the whole world. There are several file formats for such tiles. For AlpineMaps.org so-called mapbox vector tiles [4] are relevant. These are stored in mvt files. A mvt file contains several layers (e.g. hotels, shops and restaurants). For each layer it contains geographic data within the region covered by the tile.  The content of vector tiles can be visualized using a software like QGIS [5]. (The mvt files can be placed with drag & drop directly in the Layer window on the left side).

## Backround Info: PBF 
The mvt files are stored  in Protocol buffer Binary Format (PBF). This is a binary file format for OpenStreetMap data that uses Google Protocol Buffers as low-level storage. In the case of mvt files the layout of the buffer is stored in the file vector_file.proto [7].

# Tilesets
Tilesets are a collection of tiles stored in one file. From a tileset one can retrieve a vector tile. Usually tilesets provide tiles for several zoom levels starting from zoom level 0 containing one tile covering the whole world. For every zoom level z one can retrieve vector tiles with index x,y. Which region of the world is covered by a tile depends on the tile format. For example at zoom level 1 the world could be covered by two tiles with indices (0,0) and (0,1) or by two tile with indices (0,0) and (1,0). 
EAWS regions are stored as a tileset in a PMTiles file [3] from which mapbox vector tiles can be retrieved.
Usually tilesets are served by a server. A good option for serving vector tiles from a PMTiles file is Map Libre’s tile server Martin [7].

## Tileset for EAWS Regions
Avalanche.org provides all EAWS Regions on its website [1]. All these regions are stored as GeoJson files. From this collection of GeoJson files a PMTiles file is created using the software tippercanoe [2]. This file is stored on the avalanche.org website as eaws-regions.pmtiles. This tileset allows 10 zoom levels. The vector tiles from the tile set also have a so-called extent that measure the resolution of the tile. This is usually 4096x4096. The eaws-regions.pmtiles contains all current regions and outdated regions .

# Retrieving vector tiles from a Tileset with Martin Server
After downloading the file eaws-regions.pmtiles  from avalanches.org [1] and placing it in the same folder as the martin server one starts the Martin Server with .\martin.exe .\eaws-regions.pmtiles. It runs at localhost:3000. 

## From the browser:
One can access general information about the contained data from localhost:3000/catalog. 
EAWS vector tiles have three layers: "micro-regions", "micro-regions_elevation" and "outline". The relevant layer is "micro-regions". 
localhost:3000/eaws-regions displays all data stored in this layer.
localhost:3000/eaws-regions/z/x/y downloads a mvt file at zoom level z, index (xy)
## From QGIS:
One can also use QGIS to access the tileset from the martin server.
Go to Layer >> Datenquellenverwaltung >> Vektorkachel >> neu >> allgemeine Verbindung.
Activate „Dienst“ and create a new connection where you set
URL = http://localhost:3000/eaws-regions/{z}/{x}/{y},
Min Zoom = 0 Max zoom = 10.
Name = Martin EAWS vector tiles
In the main window on the left is the browser menu. There you can now click on “Vector Tiles” and then “Martin EAWS vector tiles” to display all the EAWS regions.

# Parsing  a vector tile file with EAWS regions
A downloaded EAWS mvt file contains the layer  "micro-regions". This layer conatins several features, each feature representing one micro region.
Every feature (micro region) contains at least one property and one geometry, the latter one holding the vertices of the region's boundary polygon.
Every Property has an id. There is at least one property with its id holding the name-string of the region (e.g. "FR-64"). Further properties can exist with ids (all string) "alt-id", "start_date" or "end_date".

[1] https://regions.avalanches.org/index.html  
[2] https://github.com/mapbox/tippecanoe  
[3] https://docs.protomaps.com/pmtiles  
[4] https://docs.mapbox.com/data/tilesets/guides/vector-tiles-introduction/  
[5] https://ww.qgis.org  
[6] https://maplibre.org/martin/  
[7] https://github.com/mapbox/vector-tile-spec/blob/master/2.1/vector_tile.proto  
