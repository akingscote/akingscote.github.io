---
title: "Setting Up a Tegola Server with OpenStreetMap data and custom points"
date: "2020-05-27"
categories: 
  - "development"
  - "miscellaneous"
  - "play"
tags: 
  - "gis"
  - "go"
  - "points"
  - "postgit"
  - "postgresql"
  - "tileserver"
coverImage: "Openstreetmap_logo.png"
---

This post details the steps required to run your own tile server and import data from open street map. I'll also show you how to import your own points data, so you can plot layer data into the server. This post is a prerequisite for a project i have and ill post more information about that as it matures.

The final result is something like this: ![](/images/reading-station-point.png). I googled the coordinates for Reading station and plotted a point on the map. Information has automatically been collated from the OpenStreetMap data about that specific source.

There is a miriad of tileserver implementations out there, but the one that caught my eye was [tegloa](https://tegola.io/). There are some steps on the tegola website, but it seems that the site has gone into a bit of disrepair. Often on the site it refrences osm.tegola.comm which 404s for me and their example clients reference openlayer javascript CDNs that also 404. However, the source code is on [github](https://github.com/go-spatial/tegola) so i figured worst case i could take a ganders as im getting pretty confident with my Go.

In all these examples, im just using the username password combination of "ashley:ashley" and am just running things locally. So you will see hardcoded references to my passwords, dont get excited, my password isnt usually my name!

Tegola uses PostGIS which is a Geographic Information System addon for PostgreSQl. So the first dependency is PostgreSQl. My environment for all of this isUbuntu 18.04 Desktop.

Download PostgreSQL 9.7 (im not 100% sure on newer verstions).
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list' 
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add - 
sudo apt-get update -y
sudo apt-get install postgresql-9.6 -y
```
Now install PostGIS
```
sudo apt-get install postgis
```

I recommend installing pgadmin as its a great tool for debugging PostgreSQL databases.
```
sudo apt-get install pgadmin4
```
Fire it up by running the "pgadmin4" command. It will open up a tab in a browser for you.

Complete the following steps to get the database all set up before you follow the browser wizard to get pgadmin to connect.

Create a user account that isnt just the default. This will also allocate the create database permission
```
sudo -u postgres createuser ashley --createdb
```

Log in as the postgres user which will have been created during installation and update the user password.
```
sudo -u postgres psql
ALTER USER ashley WITH PASSWORD 'new_password';
```
Once you have a user with a password, you can log in with pgadmin.

Now next we need to download some map data. The obvious source is OpenStreetMap. This got a little bit confusing for me as there is a lot of different ways to get the data. I didnt want the whole world, or even a country, i just really want a county - Berkshire. If its a very small area (small town) you can use openstreetmaps export tool.

If that dosent work, they list a couple of options on the extract tool site. I used the geofabrik website as the area i wanted was too big for the online tool. Alternatively you can download planet/country but youll need a mega pc use that properly i suspect.

Geofabrik have regular pulls of the data and group it into managable sizes. Here is the data for england: [https://download.geofabrik.de/europe/great-britain/england.html](https://download.geofabrik.de/europe/great-britain/england.html)

I used the data for Berkshire - [https://download.geofabrik.de/europe/great-britain/england/berkshire.html](https://download.geofabrik.de/europe/great-britain/england/berkshire.html) Now the data is in a ".osm.pbf" file and there are a number of tools out there to convert this data into a format for PostgreSQl. Ill save you some time - use imposm3 - [https://github.com/omniscale/imposm3/releases](https://github.com/omniscale/imposm3/releases)

Download imposm3, extract the contents and add the executable to your path.

Next, create two databases. One called "gis" which will hold all of the OpenStreetMap data and another called "natural_earth" which will hold the vector graphics for actually rendering the data.
```
createdb gis
createdb natural_earth
```

Now if PostGIS is installed properly, you will be able to add some extensions to these databases.
```
sudo -u postgres psql -d gis -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;'
sudo -u postgres psql -d natural_earth -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;'
```
You may have to run the "CREATE EXTENSION" commands individually for each database.

Next, download this imposm3.json file and save it as mapping.json - [https://raw.githubusercontent.com/go-spatial/tegola-osm/master/imposm3.json](https://raw.githubusercontent.com/go-spatial/tegola-osm/master/imposm3.json)

Import the OSM data into the gis database. Make sure that the `postgis` and `hstore` extensions are enabled on the `gis` database.
```
imposm import -connection postgis://ashley:ashley@127.0.0.1/gis -mapping mapping.json -read berkshire-latest.osm.pbf -write
imposm import -connection postgis://ashley:ashley@127.0.0.1/gis -mapping mapping.json -deployproduction
```

If you have trouble with the above, you can try using osm2pgsl - [https://github.com/openstreetmap/osm2pgsql#usage](https://github.com/openstreetmap/osm2pgsql#usage) Download a tegola executable - https://github.com/go-spatial/tegola/releases Pop it in a suitable location (e.g. /home/ashley/Desktop/tile_stuff)

Install the following prereqs
```
sudo apt-get install -y curl
sudo add-apt-repository ppa:ubuntugis/ppa
sudo apt-get update
sudo apt-get install -y gdal-bin
```

Download and run this script - [https://raw.githubusercontent.com/go-spatial/tegola-osm/master/osm_land.sh](https://raw.githubusercontent.com/go-spatial/tegola-osm/master/osm_land.sh)

Download postgis_helpers.sql and install
```
wget https://raw.githubusercontent.com/go-spatial/tegola-osm/master/postgis_helpers.sql
psql -U ashley -d gis -a -f postgis_helpers.sql
```

Download postgis_index.sql and install
```
wget https://raw.githubusercontent.com/go-spatial/tegola-osm/master/postgis_index.sql
psql -U ashley -d gis -a -f postgis_index.sql
```

Download [https://raw.githubusercontent.com/go-spatial/tegola-osm/master/natural_earth.sh](https://raw.githubusercontent.com/go-spatial/tegola-osm/master/natural_earth.sh) and run it. It will download the natural_earth vectors and put them in the `natural_earth` database, so that stuff actually renders. It takes a little while to download and install.

Next we need to create a .toml file which tegola will use. Use the following .toml file which has some of the layers removed. The original can be found here https://raw.githubusercontent.com/go-spatial/tegola-osm/master/tegola.toml Obviously change the two credential blocks to match your own credentials. Save the file as "config.toml". In this config file ive stripped out the amenity and other data that isnt really that interesting. You can use the original file as a reference for other provider data.

```
[webserver]
port =  ":8080"
 
[[providers]]
name = "osm"
type = "postgis"
host = "localhost"
port = 5432
database = "gis"
user = "ashley"
password = "ashley"
 
    [[providers.layers]]
    name = "land_8-20"
    geometry_fieldname = "geometry"
    id_fieldname = "ogc_fid"
    sql = "SELECT ST_AsBinary(wkb_geometry) AS geometry, ogc_fid FROM land_polygons WHERE wkb_geometry && !BBOX!"
 
    # Water
    [[providers.layers]]
    name = "water_areas"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, type, area FROM osm_water_areas WHERE type IN ('water', 'pond', 'basin', 'canal', 'mill_pond', 'riverbank', 'dock') AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "water_areas_gen0"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, type, area FROM osm_water_areas_gen0 WHERE type IN ('water', 'pond', 'basin', 'canal', 'mill_pond', 'riverbank') AND area > 1000000000 AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "water_areas_gen0_6"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, type, area FROM osm_water_areas_gen0 WHERE type IN ('water', 'pond', 'basin', 'canal', 'mill_pond', 'riverbank') AND area > 100000000 AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "water_areas_gen1"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, type, area FROM osm_water_areas_gen1 WHERE type IN ('water', 'pond', 'basin', 'canal', 'mill_pond', 'riverbank') AND area > 1000 AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "water_lines"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, type FROM osm_water_lines WHERE type IN ('river', 'canal', 'stream', 'ditch', 'drain', 'dam') AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "water_lines_gen0"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, type FROM osm_water_lines_gen0 WHERE type IN ('river', 'canal') AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "water_lines_gen1"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, type FROM osm_water_lines_gen1 WHERE type IN ('river', 'canal', 'stream', 'ditch', 'drain', 'dam') AND geometry && !BBOX!"
 
    # Land Use
    [[providers.layers]]
    name = "landuse_areas"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, class, type, area FROM osm_landuse_areas WHERE geometry && !BBOX!"
 
    [[providers.layers]]
    name = "landuse_areas_gen0"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, class, type, area FROM osm_landuse_areas_gen0 WHERE type IN ('forest','wood','nature reserve', 'nature_reserve', 'military') AND area > 1000000000 AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "landuse_areas_gen0_6"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, class, type, area FROM osm_landuse_areas_gen0 WHERE type IN ('forest','wood','nature reserve', 'nature_reserve', 'military') AND area > 100000000 AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "landuse_areas_gen1"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, class, type, area FROM osm_landuse_areas_gen1 WHERE geometry && !BBOX!"
 
    # Transport
 
    [[providers.layers]]
    name = "transport_lines_gen0"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, type, tunnel, bridge, ref FROM osm_transport_lines_gen0 WHERE type IN ('motorway','trunk','motorway_link','trunk_link','primary') AND tunnel = 0 AND bridge = 0  AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "transport_lines_gen1"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, ref, class, type FROM osm_transport_lines_gen1 WHERE type IN ('motorway', 'trunk', 'primary', 'primary_link', 'secondary', 'motorway_link', 'trunk_link') AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "transport_lines_11-12"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, ref, class, type, tunnel, bridge, access, service FROM osm_transport_lines WHERE type IN ('motorway', 'motorway_link', 'trunk', 'trunk_link', 'primary', 'primary_link', 'secondary', 'secondary_link', 'tertiary', 'tertiary_link', 'rail', 'taxiway', 'runway', 'apron') AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "transport_lines_13"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, ref, class, type, tunnel, bridge, access, service FROM osm_transport_lines WHERE type IN ('motorway', 'motorway_link', 'trunk', 'trunk_link', 'primary', 'primary_link', 'secondary', 'secondary_link', 'tertiary', 'tertiary_link', 'rail', 'residential', 'taxiway', 'runway', 'apron') AND geometry && !BBOX!"
 
    [[providers.layers]]
    name = "transport_lines_14-20"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, ref, class, type, tunnel, bridge, access, service FROM osm_transport_lines WHERE geometry && !BBOX!"
 
    # Buildings
    [[providers.layers]]
    name = "buildings"
    geometry_fieldname = "geometry"
    id_fieldname = "osm_id"
    sql = "SELECT ST_AsBinary(geometry) AS geometry, osm_id, name, nullif(as_numeric(height),-1) AS height, type FROM osm_buildings WHERE geometry && !BBOX!"
 
#   Natural Earth
[[providers]]
name = "ne"
type = "postgis"
host = "localhost"
port = 5432
database = "natural_earth"
user = "ashley"
password = "ashley"
 
    [[providers.layers]]
    name = "ne_110m_land"
    geometry_fieldname = "geometry"
    id_fieldname = "ogc_fid"
    sql = "SELECT ST_AsBinary(wkb_geometry) AS geometry, ogc_fid, featurecla, min_zoom FROM ne_110m_land WHERE wkb_geometry && !BBOX!"
 
    [[providers.layers]]
    name = "ne_50m_land"
    geometry_fieldname = "geometry"
    id_fieldname = "ogc_fid"
    sql = "SELECT ST_AsBinary(wkb_geometry) AS geometry, ogc_fid, featurecla, min_zoom FROM ne_50m_land WHERE wkb_geometry && !BBOX!"
 
    [[providers.layers]]
    name = "ne_10m_land"
    geometry_fieldname = "geometry"
    id_fieldname = "ogc_fid"
    sql = "SELECT ST_AsBinary(wkb_geometry) AS geometry, ogc_fid, featurecla, min_zoom FROM ne_10m_land WHERE wkb_geometry && !BBOX!"
 
    [[providers.layers]]
    name = "ne_10m_roads_3"
    geometry_fieldname = "geometry"
    id_fieldname = "ogc_fid"
    sql = "SELECT ST_AsBinary(wkb_geometry) AS geometry, ogc_fid, name, min_zoom, min_label, type, label FROM ne_10m_roads WHERE min_zoom < 5 AND type <> 'Ferry Route' AND wkb_geometry && !BBOX!"
 
    [[providers.layers]]
    name = "ne_10m_roads_5"
    geometry_fieldname = "geometry"
    id_fieldname = "ogc_fid"
    sql = "SELECT ST_AsBinary(wkb_geometry) AS geometry, ogc_fid, name, min_zoom, min_label, type, label FROM ne_10m_roads WHERE min_zoom <= 7  AND type <> 'Ferry Route' AND wkb_geometry && !BBOX!"
 
 
[[maps]]
name = "osm"
attribution = "OpenStreetMap"
center = [-0.98403, 51.44507, 13.0] #Reading
 
    # Land Polygons
    [[maps.layers]]
    name = "land"
    provider_layer = "ne.ne_110m_land"
    min_zoom = 0
    max_zoom = 2
 
    [[maps.layers]]
    name = "land"
    provider_layer = "ne.ne_50m_land"
    min_zoom = 3
    max_zoom = 4
 
    [[maps.layers]]
    name = "land"
    provider_layer = "ne.ne_10m_land"
    min_zoom = 5
    max_zoom = 7
 
    [[maps.layers]]
    name = "land"
    provider_layer = "osm.land_8-20"
    dont_simplify = true
    min_zoom = 8
    max_zoom = 20
 
    # Land Use
    [[maps.layers]]
    name = "landuse_areas"
    provider_layer = "osm.landuse_areas_gen0"
    min_zoom = 3
    max_zoom = 5
 
    [[maps.layers]]
    name = "landuse_areas"
    provider_layer = "osm.landuse_areas_gen0_6"
    min_zoom = 6
    max_zoom = 9
 
    [[maps.layers]]
    name = "landuse_areas"
    provider_layer = "osm.landuse_areas_gen1"
    min_zoom = 10
    max_zoom = 12
 
    [[maps.layers]]
    name = "landuse_areas"
    provider_layer = "osm.landuse_areas"
    min_zoom = 13
    max_zoom = 20
 
    # Water Areas
    [[maps.layers]]
    name = "water_areas"
    provider_layer = "osm.water_areas_gen0"
    min_zoom = 3
    max_zoom = 5
 
    [[maps.layers]]
    name = "water_areas"
    provider_layer = "osm.water_areas_gen0_6"
    min_zoom = 6
    max_zoom = 9
 
    [[maps.layers]]
    name = "water_areas"
    provider_layer = "osm.water_areas_gen1"
    min_zoom = 10
    max_zoom = 12
 
    [[maps.layers]]
    name = "water_areas"
    provider_layer = "osm.water_areas"
    min_zoom = 13
    max_zoom = 20
 
    # Water Lines
    [[maps.layers]]
    name = "water_lines"
    provider_layer = "osm.water_lines_gen0"
    min_zoom = 8
    max_zoom = 12
 
    [[maps.layers]]
    name = "water_lines"
    provider_layer = "osm.water_lines_gen1"
    min_zoom = 13
    max_zoom = 14
 
    [[maps.layers]]
    name = "water_lines"
    provider_layer = "osm.water_lines"
    min_zoom = 15
    max_zoom = 20
 
    # Transport Lines (Roads, Rail, Aviation)
    [[maps.layers]]
    name = "transport_lines"
    provider_layer = "ne.ne_10m_roads_3"
    min_zoom = 3
    max_zoom = 4
 
    [[maps.layers]]
    name = "transport_lines"
    provider_layer = "ne.ne_10m_roads_5"
    min_zoom = 5
    max_zoom = 6
 
    [[maps.layers]]
    name = "transport_lines"
    provider_layer = "osm.transport_lines_gen0"
    min_zoom = 7
    max_zoom = 8
 
    [[maps.layers]]
    name = "transport_lines"
    provider_layer = "osm.transport_lines_gen1"
    min_zoom = 9
    max_zoom = 10
 
    [[maps.layers]]
    name = "transport_lines"
    provider_layer = "osm.transport_lines_11-12"
    min_zoom = 11
    max_zoom = 12
 
    [[maps.layers]]
    name = "transport_lines"
    provider_layer = "osm.transport_lines_13"
    min_zoom = 13
    max_zoom = 13
 
    [[maps.layers]]
    name = "transport_lines"
    provider_layer = "osm.transport_lines_14-20"
    min_zoom = 14
    max_zoom = 20
 
    # Buildings
    [[maps.layers]]
    name = "buildings"
    provider_layer = "osm.buildings"
    min_zoom = 14
    max_zoom = 20
```

Then start the tegola server up with: `./tegola serve --config=config.toml`

If you navigate to localhost:8080 in a browser, you will see Reading being rendered.
![](/images/reading-Map.png)

Note - you wont ahave the example layer, thats coming now....

So PostGIS uses "geometry" data rather than using just latitude and longitude. So any data you want to plot you'll need to have its "geometry" reference. However, there is a nifty way that you can derive the geometry data from lat/long coordinates.

The aim here is to create a database and table, figure out the schema, manually insert some data and figure out the .toml config to get it to render. Firstly, create a database. Im calling mine "public_data" which is a reference to a future project
```
createdb public_data
```

Add the extensions to this database
```
sudo -u postgres psql -d public_data -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;'
```

Log into the `public_data` database and create a table.
```
sudo -u postgres psql public_data
```

Copy this SQL to create a table called "example". Notice that the geom field is of type geometry which has been made available to PostgreSQL via the PostGIS extension. The 4326 dictates that the data we are using is in OpenStreetMap format.
```
CREATE TABLE example ( 
  id  INTEGER,
  name VARCHAR,
  latitude FLOAT,
  longitude FLOAT,
  geom geometry(Point, 4326),
  tags HSTORE,
  CONSTRAINT example_pky PRIMARY KEY (id)
);
```

Make sure that the permissions are set. A good test to check that it is working is to View data (not just columns) in pgadmin
```
GRANT CONNECT ON DATABASE public_data TO ashley;
GRANT ALL PRIVILEGES ON TABLE example TO ashley;
```

Now insert a record with some lat and long data
```
INSERT INTO example(id, name, latitude, longitude, tags) VALUES (1, 'reading station', 51.455893, -0.971474, '"name" => "reading station"');
```

Now here is the magic, once your data is in your table, you can figure out the geometry by using a number of PostGIS function. use the following command to update the geom field for all rows in the example table.
```
UPDATE example SET geom = ST_MakeValid(ST_SetSRID(ST_MakePoint(longitude, latitude), 4326));
```

The MakeValid function may not be entirely necessary, but i figured its not doing any harm. The MakePoint can be replaced with similar functions for making lines or polygons rather than just a dump point.

Now here is a little terminal dump, which shows the data being created.Note this was taken before i had inserted htype data but you get the idea.
```
public_data=# INSERT INTO example(id, name, latitude, longitude) VALUES (1, 'reading station', 51.455893, -0.971474);
INSERT 0 1
public_data=# SELECT * FROM example;
 id |      name       | latitude  | longitude | geom 
----+-----------------+-----------+-----------+------
  1 | reading station | 51.455893 | -0.971474 | 
(1 row)
 
public_data=# UPDATE example SET geom = ST_SetSRID(ST_MakePoint(longitude, latitude), 4326);
UPDATE 1
public_data=# SELECT * FROM example;
 id |      name       | latitude  | longitude |                        geom                        
----+-----------------+-----------+-----------+----------------------------------------------------
  1 | reading station | 51.455893 | -0.971474 | 0101000020E6100000FA415DA45016EFBFD8BCAAB35ABA4940
(1 row)
```

Now i have the geometry value populated, ill see if i can get it rendered on the regola server. Add the following as a provider to the `config.toml` file
```
# custom provider
[[providers]]
name = "example"
type = "postgis"
host = "localhost"
port = 5432
database = "public_data"
user = "ashley"
password = "ashley"
 
    [[providers.layers]]
    name = "example_layer"
        geometry_type = "Point"
    id_fieldname = "id"
        srid = 4326
    sql = "SELECT ST_AsBinary(geom) AS geom, name, id FROM example WHERE geom && !BBOX!"
```

Then add a layer. Note - i spent a couple of hours trying to figure out why my point wouldnt appear, turns out it had, i just couldnt see it behind the transport lines.So put thislayer definition last.
```
#my test point
[[maps.layers]]
name = "example"
provider_layer = "example.example_layer"
min_zoom = 14
max_zoom = 20
```

Stop the existing tegola process and fire another up with the updated config. You may find it easier to disable the transport lines.

You'll notice that the minimum zoom for this point is 14. So the point wont render unless you zoom in to more than level 14. You can see what level you are currently at by looking in the URL.

Here is a GIF which shows the server in action and also shows my custom data point at Reading station. You can see in the URL that the point dosent get rendered unless i zoom in past level 14 - the minimal zoom level set in the layer config. In the GIF, i click on the Inspect Features button and navigate the cursor over my point. An overlay displays the "name" field i set for the point. Its also shows other data associated with that geometry point.

![](/images/custom point rendered - test.gif)

The Tegola server is meant to be just that - the server. If you want customisation you use clients that use the tegola backend. Their website shows an example using OpenLayers but the javascript sources are outdated and need to be updated to [https://cdn.jsdelivr.net/gh/openlayers/openlayers.github.io@master/en/v6.3.1/build/ol.js](https://cdn.jsdelivr.net/gh/openlayers/openlayers.github.io@master/en/v6.3.1/build/ol.js). However, ive not really played around with that in detail as for now, the server rendering gives me what i want.

## Debug

To log into postgres `sudo -u postgres psql postgres`

Then you can SQL commands. To update a users role: `ALTER ROLE ashley WITH CREATEDB;`

To delete all from a table `truncate table example;`
