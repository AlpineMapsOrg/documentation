## Connecting and Working on TU Wien Server
The following section shortly goes over the most important command line commands you need to connect and work with the vector tile production server, that is located on the TU Wien.

NOTE: you need to be within TU Wiens network (use VPN if you are accessing it from outside)

**Connecting to the server**
```
ssh <youruser>@osm.cg.tuwien.ac.at
```

**Accessing psql database**
```
sudo -umartin psql -Umartin gis
```

**Copy files from local to server**
Execute from terminal on your local machine
```
scp *.sql <youruser>@osm.cg.tuwien.ac.at:/usr/people/<youruser>/
```

**Apply SQL files to server**
```
sudo -umartin psql -Umartin gis -f somesqlfile.sql
```

**Updating the Martin server (clears cache):**
```
cd ~martin/
sudo -umartin ./update_docker.sh
```

Additional information can be found here: https://www.cg.tuwien.ac.at/wib/index.php/Osm

