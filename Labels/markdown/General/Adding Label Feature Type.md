# Adding Label Feature Type
Before adding a new feature you should do some rudimentary research about it on https://wiki.openstreetmap.org. The OSM wiki provides an overview about different kinds of types of Points of Interest and their required and recommended tags. With all that said please be advised that often those guidelines are not followed, and you should use the tools provided to get a better overview about the data (e.g. use metahelper\_* function (see below))
## Add feature in server
1. copy the template and rename it.
2. follow the TODOs and fill it to the best of your knowledge
	1. only fill the where field for now the concrete attributes can be filled out later
	2. if you need help with the WHERE clause I advise you to use the test.sql and a simple SELECT query to find the best selection for your POIs.
3. execute the SQL (this should create a metahelper\_* class)
	1. ```psql alpinemaps -U alpine -f yourfile.sql```
4. create and execute a query with the metahelper function to get an overview of your data
	1. ```SELECT * from metahelper_<YOUR_POI>(10,10) ORDER BY entries_with_values DESC;```
	2. ```psql alpinemaps -U alpine -f test.sql > your_poi_data.csv```
5. Analyze the resulting CSV file 
6. add the properties of your choosing to the hstore (field data) and also add them at the final *\_tile function
	1. you also should use COALESCE to combine similar fields together (e.g. the link to the webcam image was distributed between 'contact:webcam', 'image', and 'contact:website')
7. finally execute the SQL again and add the final *\_tile function to the 99\_combine.sql (and execute the SQL of this file too)
8. (optional) add the tile to the config.yaml (since we are using the 99\_combine.sql this is not necessary)
9. execute the update\_docker.sh script so that all the caches are renewed, and the new tile data will be sent
## Add feature in qt project

### Add icon
The icons we are currently using are osm-carto icons (https://github.com/gravitystorm/openstreetmap-carto). Find the desired icon for the type and put the SVG into the app/icons/labels/ folder.
Since the repository only provides SVG files use the following command to create a PNG file with similar dimensions and dpi as the other icons:
```convert -density 1200 -resize 32x32 -background none peak.svg peak.png```

After this go to the nucleus/CMakeLists.txt file and add the new PNG to the list of map\_icons qt resources.
### nucleus::vector\_tiles::VectorTypeFeature.h
- add the type in the FeatureType enum
- add a new FeatureTXT* struct with the appropriate properties and create the parse, get\_feature\_data and label\_text functions
### nucleus::vector\_tiles::VectorTileManager.h
- add parser method to feature\_types\_factory
### nucleus::map\_label::LabelFactory.cpp
- add the icon to get\_label\_icons method
### nucleus::map\_label::FilterDefinitions.h
- add properties to FilterDefinitions struct
	- e.g. m\_\*\_visible and other attributes
- add QProperties for all properties you just added
### app/FilterWindow.qml
- add appropriate GUI elements to the QML that manipulate the freshly created QProperties
### nucleus::map\_label::MapLabelFilter.cpp
- add the filters of your choice to the apply\_filter method (in a similar fashion to the existing filters)
