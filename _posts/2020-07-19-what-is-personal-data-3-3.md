---
title: "What is Personal Data? (3/3)"
date: "2020-07-19"
categories: 
  - "miscellaneous"
  - "security"
tags: 
  - "database"
  - "gis"
  - "golang"
  - "opendata"
  - "osm"
  - "postgres"
  - "privacy"
  - "security"
  - "tegola"
coverImage: "logo.png"
---

After consolidating just those 3 data sources, i have a HUGE amount on information on the residents of Reading. Obviously the results are only as good as the data. The companies house data is making some big assumptions, but its useful as another piece of information to piece together the picture.

Install the following prereqs:
```
sudo apt-get install -y curl
sudo add-apt-repository ppa:ubuntugis/ppa
sudo apt-get update
sudo apt-get install -y gdal-bin
```

Run [this script](https://raw.githubusercontent.com/go-spatial/tegola-osm/master/osm_land.sh) to install the Open Street Maps land polygons.

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

Download [natural_earth](https://raw.githubusercontent.com/go-spatial/tegola-osm/master/natural_earth.sh) and run it It will download the natural_earth vectors and put them in the `natural_earth` database, so that stuff actually renders.

Make sure you have the GIS extensions enabled on the database youre using for the public data.
```
sudo -u postgres psql -d public_data -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;'
```

Here is an example of the table schema i used:
```
sudo -u postgres psql public_data
 
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

Now make the `geometry` field be the output of conversion of the latitude and longitude fields. Here im using the Point geometry type
```
UPDATE example SET geom = ST_MakeValid(ST_SetSRID(ST_MakePoint(longitude, latitude), 4326));
```
This query is magic and will convert all lat/long fields into the geometry type required by tile servers. The 4326 signifies the specific type of geometry required by tegola.

Here is an example before/after:
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
Now i have the geometry value populated, ill see if i can get it rendered on the regola server.

I installed pgadmin and connected it to the postgresql database as its much easier to manage the data with a GUI.
![](/images/pgadmin.png)

If you run into a problem, you can delete everything from a table in postgres using truncate `truncate table example;`

So after i uploaded my data from my sqlite database into postgres (companieshouseofficers, landregistry & openregister tables), i just needed to update config.toml file so that tegola can render the layers seperately.

Now obviously, i could do some SQL table merges and cross reference on names/addresses, but i dont really need to do that. Im not actually interested in finding specific information on a person, i just want to demonstrate the vast information available.

Again, i feel i need to reiterate something: this is open data! Ive sourced this with minimal effort - in the scope of hours. Its not worth thinking about what any sort of motivated individual can source.

Im not interested in plotting every field available on each data source, just a select few. For the open register, im extracting just the names (and geometry obviously) For the companies house officers, im extracting name, company name and geometry. For the pricePaid (land registry) data, im extracting price paid, transaction date, address and geometry.

For completeness, here is my config.toml file:
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
user = "<username>"
password = "<password>"
 
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
 
# custom provider
[[providers]]
name = "public_data"
type = "postgis"
host = "localhost"
port = 5432
database = "public_data"
user = "ashley"
password = "ashley"
 
    [[providers.layers]]
    name = "openregister"
    geometry_type = "Point"
    id_fieldname = "id"
    srid = 4326
    sql = "SELECT ST_AsBinary(geom) AS geom, name, id FROM openregister WHERE geom && !BBOX!"
 
    [[providers.layers]]
    name = "companiesHouseOfficers"
    geometry_type = "Point"
    id_fieldname = "id"
    srid = 4326
    sql = "SELECT ST_AsBinary(geom) AS geom, name, company_name, id FROM companiesHouseOfficers WHERE geom && !BBOX!"
 
 
    [[providers.layers]]
    name = "pricePaid"
    geometry_type = "Point"
    id_fieldname = "id"
    srid = 4326
    sql = "SELECT ST_AsBinary(geom) AS geom, value, transaction_date, address1, id FROM landRegistry WHERE geom && !BBOX!"
 
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
 
    [[maps.layers]]
    name = "openregister"
    provider_layer = "public_data.openregister"
    min_zoom = 14
    max_zoom = 20
 
    [[maps.layers]]
    name = "companiesHouseOfficers"
    provider_layer = "public_data.companiesHouseOfficers"
    min_zoom = 14
    max_zoom = 20
 
    [[maps.layers]]
    name = "pricePaid"
    provider_layer = "public_data.pricePaid"
    min_zoom = 14
    max_zoom = 20
```

Notice that ive set the zoom layers so something sensible, as i dont want all data points to render immediately. When i zoom in past level 14 on a specific area, thats when my data comes into life.

So now, lets fire up the server. `./tegola serve --config=config.toml`

Open up a browser on localhost:8080 and i can see that the Reading roads have rendered!
![](/images/reading-tegola.png)

The second i zoom in, i can see a huge amount of information.
![](/images/zoom.png)

In the interest of complete visibilty, i goofed up. For some reason, i tried sorted the data in excel for the land registry data and i ended up completely misalligning the data columns for latitude. Rather than re-running the entire dataset (the HUGE dataset) through the google API again (i'd have to sign up for a free trial) i ended up just using a smaller dataset that i had saved from a dry run. So for the land registry data, i _only_ am using 17,000 records (as opposed to 158,225 initially retrieved). So i am plotting barely 10% of the land registry data i could have - so there should be a LOT more yellow dots.

![](/images/openreg-1.png)

By using the Inspect Features utility at the bottom, i can hover over a data point and see the information ive extracted by the query. Here i have deselected companiesHouseOfficers and pricePaid and just left the open register data on. Ive zoomed in on a random area near the town center. When you look at data in a database or spreadsheet, i find that it dosent really mean much to you. 1000 records in a database is kind of difficult to realise just how powerful that information is. By plotting on a map, I feel you can get more of a feel for what that data means and how powerful that information is. Ive purposely not shown the above people specific address on the screenshot, but the point is I could have. I have that data as it is free and open. Again, the whole point in this series is to try and understand whether or not this IS private information? Evidently - to these people its not.

Ive put together a short video capture to show you around the dataset ive captured. [https://youtu.be/-lDb1QhGC1g](https://youtu.be/-lDb1QhGC1g)

Whats really cool is that the address conversion to latitude/longitude (via google api) to geometry (via SQL query) has worked brilliantly! In the video, you can see that i scroll the cursor up and down a street and the house numbers increase/decrease as expected. Its managed to plot the data quite accurately with nothing but an address!

Upon reflection, i wish i did re-capture all of that land registry data as it would have shown much more data points, however i think ive proved my point. If i had the time, i'd love to pursue this project further and include more datasets and even perform some analysis on the data.

I think that your address cannot be personal information. Whether you know it or not, your address and name association is stored on hundreds of servers and that information is not hashed (only bank information/passwords are typically). I dont believe that every company that has your information has your best interests, or even appropriate security controls. If you expect to keep that information private, its never going to happen. I feel the same way about email addresses. The issue is that you can derive a lot of secondary information about an invidividual from their address and email address.

This could go down a huge rabbit hole, but i think ill leave it there.
