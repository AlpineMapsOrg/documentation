## Vector Tile Types
Currently 4 different types of POIs are being provided to the end user: mountain peaks, cities, mountain cottages and webcams. As of the moment of this writing the tile server contains 15.702 peaks, 21.614 cities, 1.687 webcams, 418 cottages. All POIs are located within (or near) Austria and were preprocessed from the [OpenStreetMap](https://www.openstreetmap.org) dataset (with a partial exception for the webcam data (see [[Vector Tile Types#Webcam|Webcam type]] for more details about this exception)). 
### General Attributes
All the different types contain general attributes that are shared among them. The shared attributes are the OSM identifier, a name, longitude, latitude, importance and importance metric.
While the former attributes are self-explanatory let us explain the importance metric and the importance attributes with a bit more detail.

If each tile would provide all the POIs that it contains to the user, the file sizes for higher zoom levels would be far too large and the visualization of the labels far too bloated and confusing. We therefore limit the amount of features per tile to four per feature type. This means on the other hand that we have to decide which 4 POIs to choose to visualize for any given zoom level. We therefore included an importance metric attribute to the dataset from OSM. This attribute differs between types but is defined as a number. A higher number means that it is more likely to appear on a tile (e.g. for peaks we prefer POIs with higher elevations).
The importance attribute is later calculated from this importance metric. It measures the distance between itself and a POI with a higher importance metric than itself. This distance is then stored as a value between 0 and 1. If no POI has been found that has a higher importance metric than itself within a certain search radius, the importance is set to 1.

### How to find attributes provided by OSM
The data provided from OSM can be somewhat confusing and overwhelming. Most of the attributes are stored in an hstore field named tags as key/value pairs. One feature might have stored 20 different attributes while another only holds the minimal amount of attributes. Furthermore since the OSM dataset is mostly provided by individual contributors that may or may not be familiar with the specifications provided in the [OSM Wiki](https://wiki.openstreetmap.org), some fields can often be filled in incorrectly. 

In order to understand all the individual keys that a certain feature type possesses, we created *metahelper* PostgreSQL functions. Those functions traverse the whole dataset limited by the feature type and gives the developer a list of all available keys, a number of how many features are using this key and a number of individual values that certain keys possess. Additionally it provides either all individual values per key (if the number of values is short enough) or a small sub sample of those values to get a better feeling about the data stored there.

This way the developer is able to make decisions about which attributes are interesting to convey to the final application. Additionally it also provides the means to decide which attributes can be merged (e.g. if the OSM dataset was filled in incorrectly).
### Peaks
The attributes we decided to provide for the peaks feature type were the following:

**wikipedia**
&nbsp;&nbsp;&nbsp;&nbsp; Link to Wikipedia
**wikidata**
&nbsp;&nbsp;&nbsp;&nbsp; Wikidata identifier (e.g. Q123456)
**importance\_osm**
&nbsp;&nbsp;&nbsp;&nbsp; This importance value is separate from our own. The main reason we didn't use this for our importance determination was because only a few peaks contain this information.
**prominence**
&nbsp;&nbsp;&nbsp;&nbsp; The prominence of the mountain peak. This could have also been an alternative importance metric, but similar to the above, only a small subset of peaks contain this metric.
**summit\_cross**
&nbsp;&nbsp;&nbsp;&nbsp; Stores whether or not the peak contains a cross.
&nbsp;&nbsp;&nbsp;&nbsp; Possible values for this attribute are: no, pole, separate, yes, NULL
**summit\_register**
&nbsp;&nbsp;&nbsp;&nbsp; Stores whether or not the peak contains a register.
&nbsp;&nbsp;&nbsp;&nbsp; Possible values for this attribute are: no, separate, yes, NULL
**ele**
&nbsp;&nbsp;&nbsp;&nbsp; The elevation of this peak in meters. Some features also contained the meter suffix " m" (e.g. 1234 m). Other features also provided the data as a floating point number. We therefore needed to clean up this data by parsing only the numbers provided.

The elevation attribute was also used as the importance metric for this type.


### Cities
The attributes we decided to provide for the cities feature type were the following:

**wikipedia**
&nbsp;&nbsp;&nbsp;&nbsp; Link to Wikipedia
**wikidata**
&nbsp;&nbsp;&nbsp;&nbsp;Wikidata identifier (e.g. Q123456)
**population**
&nbsp;&nbsp;&nbsp;&nbsp; The number of people living in this city. This attribute is encoded as an integer.
**place**
&nbsp;&nbsp;&nbsp;&nbsp;  The size of the City as a semantic term. Possible values are: City, Town, Village, Hamlet
**population\_date**
&nbsp;&nbsp;&nbsp;&nbsp; The date from when the population data was taken.
**population\_source**
&nbsp;&nbsp;&nbsp;&nbsp; The source from where the population data was taken from.
**postal\_code**
&nbsp;&nbsp;&nbsp;&nbsp; The postal code of this city.
**website**
&nbsp;&nbsp;&nbsp;&nbsp; The main website that belongs to this city.

The population attribute was also used as the importance metric for this type.

### Cottages
The attributes we decided to provide for the cottages feature type were the following:

**wikipedia**
&nbsp;&nbsp;&nbsp;&nbsp; Link to Wikipedia
**wikidata**
&nbsp;&nbsp;&nbsp;&nbsp; Wikidata identifier (e.g. Q123456)
**description**
&nbsp;&nbsp;&nbsp;&nbsp; A short description about this feature.
**capacity**
&nbsp;&nbsp;&nbsp;&nbsp; How many people this cottage can contain.
**opening\_hours**
&nbsp;&nbsp;&nbsp;&nbsp; The opening hours of the cottage (if applicable).
**shower**
&nbsp;&nbsp;&nbsp;&nbsp; Whether or not the cottage contains a shower. Possible values are: fee, no, yes, NULL
**phone**
&nbsp;&nbsp;&nbsp;&nbsp; A Phone number to be able to contact the owners of the cottage.
**email**
&nbsp;&nbsp;&nbsp;&nbsp; An E-mail address to be able to contact the owners of the cottage.
**website**
&nbsp;&nbsp;&nbsp;&nbsp; A Website to get more information about the feature.
**internet\_access**
&nbsp;&nbsp;&nbsp;&nbsp; This attribute describes whether or not the cottage also provides an internet connection. Possible values are: no, wlan, yes, NULL
**addr\_city**, **addr\_street**, **addr\_postcode**, **addr\_housenumber**
&nbsp;&nbsp;&nbsp;&nbsp; Those attributes describe the address of the cottage.
**operator**
&nbsp;&nbsp;&nbsp;&nbsp; This attribute provides the name of the operator of the cottage.
**type**
&nbsp;&nbsp;&nbsp;&nbsp; Describes what kind of cottage it is. There are currently two different values: alpine\_hut and wilderness\_hut.
**access**
&nbsp;&nbsp;&nbsp;&nbsp; This attribute describes the access to the cottage.
**ele**
&nbsp;&nbsp;&nbsp;&nbsp; The elevation of this cottage in meters.

As there is no one single attribute which describes a cottage as being more important than another cottage, we are using the amount of attributes that a cottage possesses as the importance metric. Our reasoning being that a cottage that has more attributes on the OSM dataset, is more likely to also be more correct and current and should therefore be prioritized for visualization on the map.

### Webcam
The attributes we decided to provide for the webcam feature type were the following:

**camera\_type**
&nbsp;&nbsp;&nbsp;&nbsp; The kind of camera that is used to capture the image. Possible values for this attribute include: dome, fixed, panning, panorama, NULL
**direction**
&nbsp;&nbsp;&nbsp;&nbsp; This attribute is a value from 0-360 and describes the direction in which the camera is pointed at.
**surveillance\_type**
&nbsp;&nbsp;&nbsp;&nbsp; This attribute describes the kind of image the camera captures. Possible values for this attribute include: indoor, maxspeed, outdoor, public, traffic, webcam, NULL
**surveillance\_zone**
&nbsp;&nbsp;&nbsp;&nbsp; Similar to surveillance type it also further captures which kind of image the camera captures. Unfortunately we found out that some features were filled in incorrectly. Possible values for this attribute include: area, atm, building, parking, public, station, street, town, traffic, NULL
**image**
&nbsp;&nbsp;&nbsp;&nbsp; The URL that provides the actual image (Can be either directly to a JPG file or to a Website which shows the image).
**description**
 &nbsp;&nbsp;&nbsp;&nbsp; A short description that describes the webcam and what it captures.
**ele**
&nbsp;&nbsp;&nbsp;&nbsp; The elevation in meters.

The elevation attribute was also used as the importance metric for this type.

Since some webcams show images that are unusable for our application, we manually looked at all the individual website providers and handpicked a few providers which served images that are suitable. The current selection include the following websites:

http://foto-webcam.eu
http://terra-hd.de
http://livecam-hd.eu
http://landgasthof-adler.at
http://entners.at
http://bergoase.at
http://arlberghaus.com
http://webcam.nl
http://linz.it-wms.com
http://irrseewebcam.sein.at
http://wurzacher.eu
http://vorarlberg-cam.at
http://webcams-thalgau.at
http://unternehmen.ortswaerme.info
http://poestlingberg.it-wms.com

#### External webcams
Additionally to the webcam providers we mentioned above we also found that [Panomax](https://panomax.com) and [Feratel](https://www.feratel.at/) provide pretty suitable webcams. Unfortunately both providers only supplied a subset to the OSM dataset. After a quick investigation we found out that both datasets can be easily obtained using a simple web crawler. This web crawler is currently available in our [martin_config](https://github.com/AlpineMapsOrg/martin_config) GitHub repository and are available in a JSON format. The convert.py script automatically converts the individual features to SQL entries in the external\_webcams table. See the instructions in the linked GitHub repository on how to use the script. After this execute the SQL scripts in the out/ folder to insert them into the database.



