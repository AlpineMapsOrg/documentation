## Vector Tile Server Setup
The following explains the setup process from nothing to a working development environment for a vector tile server using [Martin](https://github.com/maplibre/martin). While this document can be viewed on its own it should ultimately be used as documentation to better understand how the https://github.com/AlpineMapsOrg/martin_config repository works. We therefore sometimes mention files that are present in this repository which can also help you with your own setup of your development environment (e.g. update\_docker.sh)

The setup (w/o martin) was automated in the [[https://github.com/AlpineMapsOrg/martin_config/blob/main/setup_postgres.sh|setup_postgres.sh]] script.
### 1. local install
1. install martin (either by using docker or downloading the latest binaries)
https://maplibre.org/martin/installation.html

Installing using the update\_docker.sh shell script should also work (see https://github.com/AlpineMapsOrg/martin_config)

2. install PostgreSQL:
https://www.postgresql.org/download/
e.g. for Ubuntu:
	sudo apt-get -y install postgresql

3. install PostGIS:
https://postgis.net/documentation/getting_started/#installing-postgis
e.g. for Ubuntu:
	sudo apt install postgresql-14-postgis-3

### 2. Postgres setup

**First up you should log in to postgres as the postgres user (default password: postgres)**
```
sudo -u postgres psql postgres
```

**Once you are in the postgres terminal change your password with the following command:**
```
\password postgres
```

**If you encounter a "peer authentication failed for user" error:**
you have to change password encoding in the `pg_hba.conf` file from "peer" to "md5" and restart the service `sudo systemctl restart postgresql`
(In Linux systems you can find the pg\_hba.conf with `locate pg_hba.conf`)

For additional info see: https://stackoverflow.com/a/18664239

**Setting up PostGIS db (in postgres/"psql" terminal):**
```
CREATE DATABASE alpinemaps;
CREATE USER alpine PASSWORD 'SOMEPASSWORD';
GRANT ALL PRIVILEGES ON DATABASE alpinemaps TO alpine;

# switching to alpinemaps database
\c alpinemaps
ALTER SCHEMA public OWNER TO alpine;
CREATE EXTENSION postgis;
CREATE EXTENSION hstore;

# leave psql terminal
\q 
```

**After this you can enter the postrgres psql terminal in the newly created database with the following command:**
```
psql alpinemaps -U alpine 
```

### 3. import OpenStreetMap (OSM) data

In order to import OSM data we are using osm2pgsql (https://osm2pgsql.org/doc/install.html)
For Ubuntu installing this tool is as easy as using the following command:
```
sudo apt install osm2pgsql
```

**Now download the current OSM data from Geofabrik:**
https://download.geofabrik.de/europe/austria-latest.osm.pbf

**By using the following command we extract the data from the .pbf file into our newly created database:**
```
osm2pgsql -d alpinemaps -U alpine -H localhost --password --hstore austria-latest.osm.pbf
```

**Additional details about --hstore flag:**
The hstore flag stores attributes as key/value pairs in one single database field. If this flag is not set only the necessary fields (e.g. position/type/id/name will be written to the database)

**More detailed info about osm2pgsql can be found on the following link:** \[[[References and Links#PostGIS 2014|PostGIS 2014]]\]

https://subscription.packtpub.com/book/programming/9781849518666/1/ch01lvl1sec15/importing-openstreetmap-data-with-the-osm2pgsql-command
