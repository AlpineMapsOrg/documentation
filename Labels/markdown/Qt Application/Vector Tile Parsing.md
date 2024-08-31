## Vector Tile Parsing
### Mapbox Vector Tile Library
We are using a [Vector Tile Library](https://github.com/mapbox/vector-tile) provided by [Mapbox](https://www.mapbox.com/). This library allows us to parse the [vector tile file format](https://github.com/mapbox/vector-tile-spec) (which essentially are byte files encoded and structured like [Protobuf](https://protobuf.dev/) files) into easily accessible c++ classes. The provided classes use std::variants for the individual variables. They also contain far more details than we ultimately need in our application.
### nucleus::vector\_tiles::VectorTileManager
The main entry point of the *VectorTileManager* is the static to\_vector\_tile() function. This function is called from the Scheduler.update\_gpu\_quads() method after a tile has been successfully retrieved (either from the network or from a cache storage). The to\_vector\_tile function converts the byte data (formatted using the [[Vector Tile Parsing#Mapbox vectortile format|Mapbox vector tile format]]) into a *VectorTile* object (=std::set of FeatureTXT). The method accomplishes the parsing easily using the [[Vector Tile Parsing#Mapbox Vector Tile Library|Mapbox Vector Tile Library]].

After the data has been parsed, we iterate over all possible layer names and subsequently over all available features of the vector tile. Layer names are defined on the vector tile server and in our case typically describe each individual feature type (like peaks, cities, ...). The *VectorTileManager* holds a *FEATURE\_TYPES\_FACTORY* map that translates each parsable feature type to a designated parsing method. 

Those parsing methods translate the Mapbox c++ classes into the correct *FeatureTXT* subtype (e.g. *FeatureTXTPeak*). The parsed FeatureTXT structs are finally stored into an std::set which is finally returned back to the caller of the to\_vector\_tile function.

During the parsing of the features the VectorTileManager additionally passes over the names of each individual feature and accrues a set of each character that appears. This set is later used to trigger renewals of the font texture. See [[Label Rendering#nucleus::map_label::Charset|Charset]] for more information.
### nucleus::vector\_tiles::FeatureTXT
FeatureTXT and derived FeatureTXTPeak, FeatureTXTCity, etc. types are structs that hold all the relevant data that our application needs about a specific feature.  Each struct possesses parsing methods that translate the data stored in the Mapbox classes to the correct attributes. The attributes are stored in the struct in formats that are representative of the actual data (e.g. elevation is stored as integers, while names are stored as a QString).

The parse method additionally receives the DataQuerier object as a parameter. This object allows us to retrieve height data of certain positions once they are loaded. Since the height data was retrieved simultaneously with the vector tile data this can be done for all positions within the current tile. This way we can evaluate the elevation of the labels even if no altitude information is given by the vector tile server (e.g. for cities) and position the label appropriately.

As soon as a FeatureTXT is parsed an internal id is generated. This id is mainly used for the label picker. The reason why we are creating an internal id is, that the current label picker only supports ids with up to 24 bits, while the id provided by OpenStreetMaps uses 64 bits. 24 bits is more than enough for the number of features we are currently using. But this might be a problem in the future if features of the whole world are added and more feature types are introduced.

Another method that each FeatureTXT struct possesses is the label\_text method. It can be used to return the actual text that is rendered on the map. (e.g. for the peak feature we show the name and the elevation in parentheses). 

Finally each struct also has a get\_feature\_data method. This method is used by the label picker to encapsulate what data is actually provided to the user once a label is picked (for example attributes that are only used internally are not stored). The data is essentially returned as key/value pairs. Additionally the method allows us to merge fields like address together into a more human-readable format.

![[vectortile.svg|center]]