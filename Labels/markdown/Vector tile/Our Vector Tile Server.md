## Our Vector Tile Server
Next we will explain how our tile server is configured to serve the vector tiles that the AlpineMaps application needs. The final queries and configurations can be seen on the [Martin Config repository](https://github.com/AlpineMapsOrg/martin_config) (specifically the queries folder).

After the aforementioned [[Vector Tile Server Setup#3. import OpenStreetMap (OSM) data|osm2pqsql import]] our SQL database now contains (among other things) the *planet\_osm\_point* table. This table contains id, name and coordinates for each feature. Furthermore it differentiates the different kinds of features with different fields. (e.g. for peaks the field "natural" should contain the value "peak", while for cities the field "place" should contain "city", "town", "village" or "hamlet"). In order to find out what to look for when implementing a new type you should look in the [OSM wiki](https://wiki.openstreetmap.org/) how each type is specified. Additionally each feature also contains a *tags* field with key/value pairs for additional attributes.

All different feature types are processed in a similar way to provide the final vector tile. This allowed us to create a *\_template.sql* file where new features can be easily created from. Additionally if something major changes in the steps of the preprocessor, we can easily amend other feature types again by only changing the template file and creating the features anew from the template. 
### Generating vector tiles
In order to generate vector tiles for one type we are using a multiple step preprocessing procedure. The main steps of our preprocessor are as follows:

1. create a materialized view from OSM table (view contains all attributes we want to provide and an importance metric)
2. create SQL indices for the coordinates and importance metric for quicker lookups of subsequent queries
3. create a second materialized view called distance\_\<featuretype\>
	1. In this view compare each feature importance metric with other features in a search radius and define a new field named importance with values between \[0,1\]
	2. the importance value is larger the farther away a feature is that possesses a larger importance metric than itself
	3. if no features with a bigger importance metric is found set importance to 1
	4. sort materialized view by importance and importance metric
4. create SQL indices for the coordinates for a quicker lookup of subsequent queries
5. create a [[Vector Tile Server#Functions|Martin vector tile function]] that provides the 4 first entries of the view we just created, limited by the constraints of the vector tile coordinates

The search radius is 1/3 the size of a tile with zoom level 8. Note for speeding up of the preprocessing procedure it is possible to define other zoom levels for this. In order to calculate this we created a utility table in the *1\_utilities.sql* file where all distances from zoom levels 5 to 25 are calculated.

The tile function is defined that it only provides values starting with zoom level 10 (set in the config.yaml file of the martin server). Starting from zoom level 22 all features within this tile are served without any limitations of amount.

We are using materialized views because this allows us to define indices on certain fields of those views. By using indices we improve the speed of queries significantly. This can be observed in the [[Vector Tile Server Benchmarks|benchmark]] section.

### Combining each feature
Martin would already allow us to combine each individual feature type into one vector tile by combining the features with commas in the request URL.

**Example:**
The ordinary URL for vector tiles of one type might look like: 
server.com/\<type\>/\<z\>/\<x\>/\<y\>
and for multiple types like:
server.com/\<type1\>,\<type2\>,\<type3\>/\<z\>/\<x\>/\<y\>

After discussing this we came to the conclusion that combining the types using commas is not ideal. We therefore created an additional function that can be seen in the *99\_combine.sql* file. This function merges each individual type and is available only with one simple URL.


