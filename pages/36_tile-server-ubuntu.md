---
layout: page
title: Installing the OpenStreetMap Tile Server on Ubuntu
comments: true
permalink: /tile-server-ubuntu/
sitemap: false
---

## Introduction

The following step-by-step procedure can be used to install, setup and configure all the necessary software to operate your own OpenStreetMap tile server on Ubuntu.

The OSM tile server stack is a collection of programs and libraries chained together to create a tile server. As so often with OpenStreetMap, there are many ways to achieve this goal and nearly all of the components have alternatives that have various specific advantages and disadvantages. This tutorial describes the most standard version that is also used on the main OpenStreetMap.org tile server.

It consists of the following main components:

* Mapnik
* Apache
* Mod_tile
* renderd
* osm2pgsql
* PostgreSQL/PostGIS database, to be installed locally (suggested) or remotely (might be slow, depending on the network).
* carto
* openstreetmap-carto

Mod_tile is an apache module that serves cached tiles and decides which tiles need re-rendering either because they are not yet cached or because they are outdated. Renderd provides a priority queueing system for rendering requests to manage and smooth out the load from rendering requests. Mapnik is the software library that does the actual rendering and is used by renderd.

{% include_relative _includes/update-ubuntu.md %}

## Install Mapnik library

We need to install the Mapnik library. Mapnik is used to render the OpenStreetMap data into the tiles managed by the Apache web server through *renderd* and *mod_tile*.

We report some alternative procedures to install Mapnik (in the consideration to run an updated version of Ubuntu).

### Install Mapnik library from package

Tested with Ubuntu 16.04 and Ubuntu 14.04; suggested as the preferred option to install Mapnik.

    sudo apt-get install -y git autoconf libtool libxml2-dev libbz2-dev \
      libgeos-dev libgeos++-dev libproj-dev gdal-bin libgdal1-dev g++ \
      libmapnik-dev mapnik-utils python-mapnik

This will most probably install Mapnik 2.2 on Ubuntu 14.04.3 LTS and Mapnik 3.0.9 on Ubuntu 16.04.1 LTS.

Go to [check Mapnik installation](#verify-that-mapnik-has-been-correctly-installed).

### Alternatively, install the lastest version of Mapnik from the GitHub nightly build

First, remove any other old Mapnik packages:

    sudo apt-get purge -y libmapnik* mapnik-* python-mapnik

The [nightly build from master](https://launchpad.net/~mapnik/+archive/ubuntu/nightly-trunk) is directly from the [GitHub repository]https://github.com/mapnik/mapnik/commits/master):

    sudo add-apt-repository ppa:mapnik/nightly-trunk
    sudo apt-get update
    sudo apt-get install -y git autoconf libtool libxml2-dev libbz2-dev \
      libgeos-dev libgeos++-dev libproj-dev gdal-bin libgdal1-dev g++ \
      libmapnik-dev mapnik-utils python-mapnik

This will most probably install Mapnik 2.2 on Ubuntu 14.04.3 LTS and Mapnik 3.0.12 on Ubuntu 16.04.1 LTS.

### Alternatively, install Mapnik from sources

Refer to [Mapnik Ubuntu Installation](https://github.com/mapnik/mapnik/wiki/UbuntuInstallation) to for specific documentation.

Refer to [Mapnik Releases](https://github.com/mapnik/mapnik/releases) for the latest version and changelog.

Remove any other old Mapnik packages:

    sudo apt-get purge -y libmapnik* mapnik-* python-mapnik
    sudo add-apt-repository --remove -y ppa:mapnik/nightly-trunk

Install prerequisites; first create a directory to load the sources:

    test -d ~/src || mkdir  ~/src ; cd ~/src

    sudo apt-get install -y libxml2-dev libfreetype6-dev \
      libjpeg-dev libpng-dev libproj-dev libtiff-dev \
      libcairo2 libcairo2-dev python-cairo python-cairo-dev \
      libgdal1-dev git

    sudo apt-get install -y build-essential python-dev libbz2-dev libicu-dev

We need to install [Boost](http://www.boost.org/) either from package or from source.

#### Install Boost from package

    sudo apt-get install libboost-all-dev

#### Alternatively, install the latest version of Boost from source

    sudo apt-get purge -y libboost-all-dev # remove installation from package
    cd ~/src
    wget -O boost.tar.bz2 https://sourceforge.net/projects/boost/files/latest/download?source=files
    tar xjf boost.tar.bz2
    rm boost.tar.bz2
    cd boost_*
    ./bootstrap.sh
    ./b2 stage toolset=gcc --with-thread --with-filesystem --with-python --with-regex -sHAVE_ICU=1 -sICU_PATH=/usr/ --with-program_options --with-system link=shared
    sudo ./b2 install toolset=gcc --with-thread --with-filesystem --with-python --with-regex -sHAVE_ICU=1 -sICU_PATH=/usr/ --with-program_options --with-system link=shared -d0
    sudo ldconfig && cd ~/

#### Alternatively, install Boost from Mapnik PPA

    sudo add-apt-repository ppa:mapnik/boost
    sudo apt-get update
    sudo apt-get install libboost-dev libboost-filesystem-dev libboost-program-options-dev libboost-python-dev libboost-regex-dev libboost-system-dev libboost-thread-dev 
    sudo apt-get install \
        libboost-filesystem-dev \
        libboost-program-options-dev \
        libboost-python-dev libboost-regex-dev \
        libboost-system-dev libboost-thread-dev
    sudo apt-get upgrade

#### Install HarfBuzz from source

[HarfBuzz](https://www.freedesktop.org/wiki/Software/HarfBuzz/) is an [OpenType](http://www.microsoft.com/typography/otspec/) text shaping engine.

Check the lastest version [here](https://www.freedesktop.org/software/harfbuzz/release/)

    cd ~/src
    wget https://www.freedesktop.org/software/harfbuzz/release/harfbuzz-1.3.2.tar.bz2
    tar xf harfbuzz-1.3.2.tar.bz2
    rm harfbuzz-1.3.2.tar.bz2
    cd harfbuzz-1.3.2
    ./configure && make && sudo make install
    sudo ldconfig
    cd ~/

#### Build the Mapnik library from source

    cd ~/src
    git clone https://github.com/mapnik/mapnik.git --depth 10
    cd mapnik
    git submodule update --init
    bash
    source bootstrap.sh
    ./configure && 

If you get the error "`C++ compiler does not support C++11 standard (-std=c++11), which is required. Please upgrade your compiler`", check the following (it might happen on Ubuntu 14.4):

    ./configure CXX=g++-4.8 CC=gcc-4.8 && make

Alternative procedure (only if problems in the previous one):

    cd ~/src
    git clone https://github.com/mapnik/mapnik.git --depth 10
    cd mapnik
    git submodule update --init
    ./configure
    make

Test Mapnik (without needing to install):

    make test # some test might not pass
    
Install Mapnik:

    sudo make install
    cd ~/

Python bindings are not included by default. You'll need to add those separately.

    cd ~/src
    git clone https://github.com/mapnik/python-mapnik.git
    cd python-mapnik
    sudo apt-get install python-setuptools
    sudo apt-get install python3-setuptools
    sudo apt-get install libboost-python-dev
    sudo python setup.py develop
    sudo python setup.py install

## Verify that Mapnik has been correctly installed

Report Mapnik version number:

    mapnik-config -v

Check then with Python:

    python -c "import mapnik;print mapnik.__file__"

It should return the path to the python bindings. If python replies without errors, then Mapnik library was found by Python.

## Install Apache HTTP Server

The [Apache](https://en.wikipedia.org/wiki/Apache_HTTP_Server) free open source HTTP Server is among the most popular web servers in the world. It's [well-documented](https://httpd.apache.org/), and has been in wide use for much of the history of the web, which makes it a great default choice for hosting a website.

To install apache:

    sudo apt-get install apache2 apache2-dev

To check if Apache is installed, direct your browser to your server’s IP address (eg. http://localhost). The page should display the default Apache home page. Also this command allows checking correct working:

    curl localhost| grep 'It works!'

## How to Find your Server’s IP address

You can run the following command to reveal your server’s IP address on its main Ethernet interface.

    ifconfig eth0 | grep inet | awk '{ print $2 }'

## Install Mod_tile

[Mod_tile](http://wiki.openstreetmap.org/wiki/Mod_tile) is an Apache module to efficiently render and serve map tiles for www.openstreetmap.org map using Mapnik. We can compile it from Github repository.

    cd ~/src
    git clone https://github.com/openstreetmap/mod_tile.git
    cd mod_tile
    ./autogen.sh
    ./configure
    make
    sudo make install
    sudo make install-mod_tile
    sudo ldconfig
    cd ~/

{% include_relative _includes/complete-inst-osm-carto.md cdprogram='' %}

{% include_relative _includes/install-git-nodejs.md program='OpenStreetMap Tile Server' %}

## Install carto and build the Mapnik xml stylesheet

    sudo apt-get install node-carto

    cd
    cd openstreetmap-carto
    scripts/yaml2mml.py
    carto project.mml > style.xml

{% include_relative _includes/configuration-variables.md os='Ubuntu' %}

{% include_relative _includes/firewall-postgis-inst.md port='80 and local port 443' %}














## Configure *renderd*

Next we need to plug renderd and mod_tile into the Apache webserver, ready to receive tile requests.

Continuare da qui:
https://switch2osm.org/serving-tiles/manually-building-a-tile-server-14-04/

Edit renderd config file.

sudo nano /usr/local/etc/renderd.conf

In the [default] section, change the value of XML and HOST to the following.

XML=/home/osm/openstreetmap-carto-2.41.0/style.xml
HOST=localhost

In [mapnik] section, change the value of plugins_dir.

plugins_dir=/usr/lib/mapnik/3.0/input/

Save the file.

Install renderd init script by copying the sample init script.

sudo cp mod_tile/debian/renderd.init /etc/init.d/renderd

Grant execute permission.

sudo chmod a+x /etc/init.d/renderd

Edit the init script file

sudo nano /etc/init.d/renderd

Change the following variable.

DAEMON=/usr/local/bin/$NAME
DAEMON_ARGS="-c /usr/local/etc/renderd.conf"
RUNASUSER=osm

Save the file.

Create the following file and set osm the owner.

sudo mkdir -p /var/lib/mod_tile

sudo chown osm:osm /var/lib/mod_tile

Then start renderd service

sudo systemctl daemon-reload

sudo systemctl start renderd

sudo systemctl enable renderd

Step 8: Configure Apache

Install apache web server

sudo apt install apache2

Create a module load file.

sudo nano /etc/apache2/mods-available/mod_tile.load

Paste the following line into the file.

LoadModule tile_module /usr/lib/apache2/modules/mod_tile.so

Create a symlink.

sudo ln -s /etc/apache2/mods-available/mod_tile.load /etc/apache2/mods-enabled/

Then edit the default virtual host file.

sudo nano /etc/apache2/sites-enabled/000-default.conf

Past the following line in <VirtualHost *:80>

LoadTileConfigFile /usr/local/etc/renderd.conf
ModTileRenderdSocketName /var/run/renderd/renderd.sock
# Timeout before giving up for a tile to be rendered
ModTileRequestTimeout 0
# Timeout before giving up for a tile to be rendered that is otherwise missing
ModTileMissingRequestTimeout 30

Save and close the file. Restart Apache.

sudo systemctl restart apache2

Then in your web browser address bar, type

your-server-ip/osm_tiles/0/0/0.png

You should see the tile of world map. Congrats! You just successfully built your own OSM tile server.


## Display Your Tiled Web Map

Tiled web map is also known as slippy map in OpenStreetMap terminology. There are two free and open source JavaScript map libraries you can use for your tile server: OpenLayer and Leaflet. The advantage of Leaflet is that it is simple to use and your map will be mobile-friendly.

### OpenLayer

To display your slippy map with OpenLayer, first create a web folder.

sudo mkdir /var/www/osm

Then download JavaScript and CSS from openlayer.org and extract it to the web root folder.

Next, create the index.html file.

sudo nano /var/www/osm/index.html

Paste the following HTML code in the file. Replace red-colored text and adjust the longitude, latitude and zoom level according to your needs.

<!DOCTYPE html>
<html>
<head>
<title>Accessible Map</title>
<link rel="stylesheet" href="http://your-ip/ol.css" type="text/css">
<script src="http://your-ip/ol.js"></script>
<style>
  a.skiplink {
    position: absolute;
    clip: rect(1px, 1px, 1px, 1px);
    padding: 0;
    border: 0;
    height: 1px;
    width: 1px;
    overflow: hidden;
  }
  a.skiplink:focus {
    clip: auto;
    height: auto;
    width: auto;
    background-color: #fff;
    padding: 0.3em;
  }
  #map:focus {
    outline: #4A74A8 solid 0.15em;
  }
</style>
</head>
<body>
  <a class="skiplink" href="#map">Go to map</a>
  <div id="map" class="map" tabindex="0"></div>
  <button id="zoom-out">Zoom out</button>
  <button id="zoom-in">Zoom in</button>
  <script>
    var map = new ol.Map({
      layers: [
        new ol.layer.Tile({
          source: new ol.source.OSM({
             url: 'http://your-ip/osm_tiles/{z}/{x}/{y}.png'
          })
       })
     ],
     target: 'map',
     controls: ol.control.defaults({
        attributionOptions: /** @type {olx.control.AttributionOptions} */ ({
          collapsible: false
        })
     }),
    view: new ol.View({
       center: [244780.24508882355, 7386452.183179816],
       zoom:5
    })
 });

  document.getElementById('zoom-out').onclick = function() {
    var view = map.getView();
    var zoom = view.getZoom();
    view.setZoom(zoom - 1);
  };

  document.getElementById('zoom-in').onclick = function() {
     var view = map.getView();
     var zoom = view.getZoom();
     view.setZoom(zoom + 1);
  };
</script>
</body>
</html>

Save and close the file. Now you can view your slippy map by typing your server IP address in browser.

your-ip/index.html           or          your-ip





### Leaflet

Leaflet is a JavaScript library for embedding maps. It is simpler and smaller than OpenLayers.

To display your slippy map with Leftlet, first create a web folder.

    sudo mkdir /var/www/osm

You can download Leaflet (JavaScript and CSS) from its own site at leftletjs.com or better from GitHub:

    cd
    git clone https://github.com/Leaflet/Leaflet.git

Copy the dist/ directory to the place on your webserver where the embedding page will be served from, and rename it leaflet/ .

    cd Leaflet
    cp -r *

Next, create the index.html file.

sudo nano /var/www/osm/index.html

Paste the following HTML code in the file. Replace red-colored text and adjust the longitude, latitude and zoom level according to your needs.

```html
<html>
<head>
<title>My first osm</title>
<link rel="stylesheet" type="text/css" href="leaflet.css"/>
<script type="text/javascript" src="leaflet.js"></script>
<style>
   #map{width:100%;height:100%}
</style>
</head>

<body>
  <div id="map"></div>
  <script>
    var map = L.map('map').setView([53.555,9.899],5);
    L.tileLayer('http://your-ip/osm_tiles/{z}/{x}/{y}.png',{maxZoom:18}).addTo(map);
</script>
</body>
</html>
```

Save and close the file. Now you can view your slippy map by typing your server IP address in browser.

To pre-render tiles instead of rendering on the fly, use render_list command. Pre-rendered tiles will be cached in /var/lib/mod_tile directory. -z and -Z flag specify the zoom level.

render_list -m default -a -z 0 -Z 10



## Embedding Leaflet in your page

For ease of use, we’ll create a .js file with all our map code in it. You can of course put this inline in the main page if you like. Create this page in your *leaflet/* directory and call it *leafletembed.js*.

Use the following code in *leafletembed.js*:

```javascript
var map;
var ajaxRequest;
var plotlist;
var plotlayers=[];

function initmap() {
	// set up the map
	map = new L.Map('map');

	// create the tile layer with correct attribution
	var osmUrl='http://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png';
	var osmAttrib='Map data © <a href="http://openstreetmap.org">OpenStreetMap</a> contributors';
	var osm = new L.TileLayer(osmUrl, {minZoom: 8, maxZoom: 12, attribution: osmAttrib});		

	// start the map in South-East England
	map.setView(new L.LatLng(51.3, 0.7),9);
	map.addLayer(osm);
}
```

Then include it in your embedding page like this:

```html
<link rel="stylesheet" type="text/css" href="leaflet/leaflet.css" />
<script type="text/javascript" src="leaflet/leaflet.js"></script>
<script type="text/javascript" src="leaflet/leafletembed.js"></script>
```

Add an appropriately-sized *div* called ‘map‘ to your embedding page; then, finally, add some JavaScript to your embedding page to initialise the map, either at the end of the page or on an *onload* event:

    initmap();

Congratulations; you have embedded your first map with Leaflet.

## Showing markers as the user pans around the map

There are several [excellent examples on the Leaflet website](http://leaflet.cloudmade.com/examples.html). Here we’ll demonstrate one more common case: showing clickable markers on the map, where the marker locations are reloaded from the server as the user pans around.

First, we’ll add the standard AJAX code of the type you’ll have seen a thousand times before. At the top of the *initmap* function in *leafletembed.js*, add:

```javascript
// set up AJAX request
ajaxRequest=getXmlHttpObject();
if (ajaxRequest==null) {
	alert ("This browser does not support HTTP Request");
	return;
}
```

then add this new function elsewhere in *leafletembed.js*:

```javascript
function getXmlHttpObject() {
	if (window.XMLHttpRequest) { return new XMLHttpRequest(); }
	if (window.ActiveXObject)  { return new ActiveXObject("Microsoft.XMLHTTP"); }
	return null;
}
```

Next, we’ll add a function to request the list of markers (in JSON) for the current map viewport:

```
function askForPlots() {
	// request the marker info with AJAX for the current bounds
	var bounds=map.getBounds();
	var minll=bounds.getSouthWest();
	var maxll=bounds.getNorthEast();
	var msg='leaflet/findbybbox.cgi?format=leaflet&bbox='+minll.lng+','+minll.lat+','+maxll.lng+','+maxll.lat;
	ajaxRequest.onreadystatechange = stateChanged;
	ajaxRequest.open('GET', msg, true);
	ajaxRequest.send(null);
}
```

This talks to a serverside script which simply returns a JSON array of the properties we want to display on the map, like this:

```json
[{"name":"Tunbridge Wells, Langton Road, Burnt Cottage",
  "lon":"0.213102",
  "lat":"51.1429",
  "details":"A Grade II listed five bedroom wing in need of renovation."}]
```

(etc.)

When this arrives, we’ll clear the existing markers and display the new ones, creating a rudimentary pop-up window for each one:

```javascript
function stateChanged() {
	// if AJAX returned a list of markers, add them to the map
	if (ajaxRequest.readyState==4) {
		//use the info here that was returned
		if (ajaxRequest.status==200) {
			plotlist=eval("(" + ajaxRequest.responseText + ")");
			removeMarkers();
			for (i=0;i<plotlist.length;i++) {
				var plotll = new L.LatLng(plotlist[i].lat,plotlist[i].lon, true);
				var plotmark = new L.Marker(plotll);
				plotmark.data=plotlist[i];
				map.addLayer(plotmark);
				plotmark.bindPopup("<h3>"+plotlist[i].name+"</h3>"+plotlist[i].details);
				plotlayers.push(plotmark);
			}
		}
	}
}

function removeMarkers() {
	for (i=0;i<plotlayers.length;i++) {
		map.removeLayer(plotlayers[i]);
	}
	plotlayers=[];
}
```

Finally, let’s wire this into the rest of our script. After we’ve added the map in *initmap*, let’s ask for the first load of markers, and set up an event to do this every time the map is panned. Add this just at the end of *initmap*:

```javascript
	askForPlots();
	map.on('moveend', onMapMove);
}

// then add this as a new function...
function onMapMove(e) { askForPlots(); }
```











Cose da controllare:
http://tilemill-project.github.io/tilemill/docs/source/

https://blog.gravitystorm.co.uk/
https://switch2osm.org/serving-tiles/manually-building-a-tile-server-14-04/