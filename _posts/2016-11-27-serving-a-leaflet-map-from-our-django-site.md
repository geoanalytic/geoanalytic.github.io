---
layout: post
comments: true
title: Serving web maps from our django site
---

Serving web pages with Django from a spatially enabled database should allow us to easily place interactive maps on our pages, and provide intelligent spatial query capabilities at the back end.  The emphasis on interactive (or slippy) maps, means that we will have to have a javascript component to handle the client interaction.  There are [many](http://techslides.com/50-javascript-libraries-and-plugins-for-maps) javascript mapping [libraries](http://gis.stackexchange.com/questions/8032/how-do-various-javascript-mapping-libraries-compare) available today, including [OpenLayers](https://openlayers.org/) which is included with GeoDjango and [Cesium](https://cesiumjs.org/) which does 3D.  I'll demonstrate a map using OpenLayers here but then I want to focus on [Leaflet](http://leafletjs.com/) which is very popular due to its small footprint and good performance.    

This post builds on the basic django/postgis server that I documented in my earlier series:  A production ready web mapping toolkit.    

* [Part 1](https://geoanalytic.github.io/a-production-ready-web-mapping-toolkit-part-1/)   
* [Part 2](https://geoanalytic.github.io/a-production-ready-web-mapping-toolkit-part-2/)   
* [Part 3](https://geoanalytic.github.io/a-production-ready-web-mapping-toolkit-part-3/)   

The code discussed in the previous posts is available, just clone my repository:   

```
$ git clone --recursive https://github.com/geoanalytic/cookie-cutter-demo.git
```

note that the `--recursive` option is required because I am using another repository for the PostGIS container build.  You will still need to edit some of the settings and environment variables, including the host domain, before you can use the site.       

If you already have a django site that you want to serve maps from, then carry on, but if not then following the process discussed in the above posts should get you up and running with everything you need.   

-------------

# Basic Requirements   

Before we begin, let's make sure we have everything we need:   

1.  A working django website, pulling data from        
2.  A PostgreSQL/PostGIS database, with at least one table containing   
3.  [Geographic data](https://docs.djangoproject.com/en/1.10/ref/contrib/gis/model-api/).  In our simple case it will be a table of Point features.  Here is the data model:    

```python
# geodata/models.py

# -*- coding: utf-8 -*-
from __future__ import unicode_literals, absolute_import
from django.utils.encoding import python_2_unicode_compatible
from django.contrib.gis.db import models

# A simple point location with a name
@python_2_unicode_compatible
class Location(models.Model):
    name = models.CharField(max_length=50, default = 'Unknown location')
    mpoint = models.PointField()

    def __str__(self):
        return self.name
```

In order to create/read/update/delete some Location data, I register the model with the admin:   

```python
# geodata/admin.py

from django.contrib.gis import admin
from .models import Location

# subclass the GeoModelAdmin to use the locally hosted OpenLayers library
class olGeoModelAdmin(admin.GeoModelAdmin):
    openlayers_url = 'OpenLayers.js'

# subclass the OSMGeoAdmin to use the locally hosted OpenLayers library
class olOSMGeoAdmin(admin.OSMGeoAdmin):
    openlayers_url = 'OpenLayers.js'

# register an admin tool for the Location model
# admin.site.register(Location, olGeoModelAdmin)
# the OSMGeoAdmin tool uses the openstreetmap data for a nicer experience
admin.site.register(Location, olOSMGeoAdmin)
```

The bits about subclassing the OpenLayers.js library are only necessary if we are serving our site from a secure (HTTPS) server.   

# Goals and Approach   

At the end of this post, I want to have:  

*  the home page of my dev.positionbot.com site show a slippy map with markers placed from the `location` table stored in the database.     

If I get ambitious, I'd also like to have:   

* a spatial search function that will select features within a specified radius of a mouse click   
* and maybe a distance calculation performed by the database and displayed on the map  

There are several ways we could accomplish these goals, but the two I will cover here are:   

1.  The minimum required code, which means an html file which loads the (hosted) javascript and gets a geojson feed from our django site.   
2.  Using an already built django package that hooks into the admin, forms, and provides a set of configuration options that makes it possible to serve lots of custom map pages without getting our hands dirty with (too much) javascript.  

# Phase 1:  Serve a Minimal Slippy Map With OpenLayers    

Doing the bare minimum just to see how easy this is.  We will put our map on the main home page of our site using the OpenLayers library.  Right now, our basic home page is generated from the cookie-cutter-demo/templates/pages/home.html file, and looks like this:   

![_config.yml]({{ site.baseurl }}/images/post4/barehome.png)

If you followed my [earlier discussion](https://geoanalytic.github.io/a-production-ready-web-mapping-toolkit-part-2/) of setting up GeoDjango, you'll see we already have the OpenLayers library installed with slippy maps available through the admin tool.  But we don't want to expose the admin to just any user, so what do we need to do to put a map on the home page?    

## Ensure the Javascript Are Loaded   

We want to keep using the existing page templates for our HTML, so the first edit will be to the file cookie-cutter-demo/templates/base.html and add the links for the style.css and OpenLayers.js files in the indicated places.  

```html
<!-- base.html -->
<!-- ... -->
      <!-- Your stuff: Third-party javascript libraries go here -->
      <script src="{% static 'js/OpenLayers.js' %}"></script>
<!-- ... -->   
```

## Write Some Javascript To Generate The Map   

Perusing the base.html file, we see a reference a file called project.js where we are intended to place our project specific javascript.  We'll follow this structure for now but in the future we may find a need to develop a different set of templates for including our code.  If you have doubts about how this is all works, the [Django template engine documents](https://docs.djangoproject.com/en/1.10/topics/templates/) are pretty good.  We will edit the cookie-cutter-demo/static/js/project.js file and add some code:    

```javascript
var lon = -114;
var lat = 51;
var zoom = 10;
var map, layer;

function init(){
    map = new OpenLayers.Map( 'map' );
    layer = new OpenLayers.Layer.WMS( "OpenLayers WMS", 
         "http://vmap0.tiles.osgeo.org/wms/vmap0",
          {layers: 'basic'} );
    map.addLayer(layer);
    map.setCenter(new OpenLayers.LonLat(lon, lat), zoom);
    var geojson_format = new OpenLayers.Format.GeoJSON();
    var vector_layer = new OpenLayers.Layer.Vector("GeoJSON", {
        projection: "EPSG:4326",
        strategies: [new OpenLayers.Strategy.Fixed()],
        protocol: new OpenLayers.Protocol.HTTP({
            url: "geodata/data.geojson",
            format: new OpenLayers.Format.GeoJSON()
        })
    }); 
    map.addLayer(vector_layer);
//    vector_layer.addFeatures(geojson_format.read(featurecollection));
}
window.onload = init;
```
You will see some code already in the project.js file, but since it's javascript, we can just insert our new code below it.    

## Add A Map Div To The HTML   

Next, we edit the file cookie-cutter-demo/templates/pages/home.html and insert a properly named div element.   

```html
{% extends "base.html" %}

{% block content %}
<h1>Time to Boogie</h1>
<div id="map" class="smallmap" style="height:800px"></div>
{% endblock %}
```

## Add A GeoJson Feed   

Finally, we need to provide a feed for the GeoJson layer referenced in our javascript code.  The URL for our feed is given as geodata/data.geojson, so lets first hook it up.  Knowledge of the [django url mapping scheme](https://docs.djangoproject.com/en/1.10/intro/overview/#design-your-urls) goes a long way here, but basically all requests get routed through the file cookie-cutter-demo/config/urls.py which can be passed on to other bits of code.  We will direct any requests to `geodata` to be passed to our geodata app like so:    

```python
# config/urls.py

# -*- coding: utf-8 -*-
# ...

urlpatterns = [
# ...

    url(r'^geodata/', include('geodata.urls')),
]

# ...
```

We also need to create a the referenced file geodata/urls.py as:

```python
# geodata/urls.py

# -*- coding: utf-8 -*-
from __future__ import absolute_import, unicode_literals

from django.conf.urls import url

from . import views

urlpatterns = [
    url(r'^data.geojson$', views.geojsonFeed, name='data'),
]

```

Finally, we create a view called geojsonFeed in our geodata/views.py file:   

```python
# geodata/views.py
from django.shortcuts import render
from django.http import HttpResponse
from django.core.serializers import serialize

from .models import Location

# Geojson serializer
def geojsonFeed(request):
    return HttpResponse(serialize('geojson', Location.objects.all(),fields=('name',)))
```

That should do it, our home page now looks like this:   

![_config.yml]({{ site.baseurl }}/images/post4/openlayers.png)

Not a pretty map, but a bit of mousing will confirm it is indeed responsive.  We can access the admin site and add some locations that should then show up on our front page.    

![_config.yml]({{ site.baseurl }}/images/post4/openlayers_points.png)

Well, it works, and I have no doubt that with some judicious tweaking of the various html, css, and javascript we could achieve a satisfactory presentation.  However, and this is one of my major misgivings about GeoDjango, we are using a deprecated verion of OpenLayers (2.13) rather than the [current version](http://openlayers.org/).  So I won't take this any further.  Let's see if we can get the same result using Leaflet.   

# Phase 2:  Serve A Minimal Slippy Map With Leaflet   

Here, the requirements are very similar to those for OpenLayers:   

1. Ensure the css and js files are loaded   
2. Add our custom javascript to draw the map   
3. Add a <div> element to the html    
4. Serve a GeoJSON feed from our django site    

The first requirement is pretty easy.  The latest [leaflet libraries](http://leafletjs.com/download.html) are hosted on a content delivery network, so we edit our cookie-cutter-demo/templates/base.html file and insert the references.  Note the css goes in the <head> section while the js goes at the end of <body>.   

```html
<!-- base.html -->

<!-- ... -->
    <!-- Your stuff: Third-party CSS libraries go here -->
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.0.2/dist/leaflet.css" />
<!-- ... -->

      <!-- Your stuff: Third-party javascript libraries go here -->
      <script src="https://unpkg.com/leaflet@1.0.2/dist/leaflet.js"></script>
      
<!-- ... -->      
```

Then we edit the cookie-cutter-demo/static/js/project.js, replacing the OpenLayers stuff we wrote before with the following:  

```javascript
// project.js
//...

var dataurl = 'geodata/data.geojson';
var lon = -114;
var lat = 51;
var zoom = 10;
var map, layer;

map = new L.map('map').setView([lat, lon], zoom);
L.tileLayer('http://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    maxZoom: 19,
}).addTo(map);


// Download GeoJSON data with Ajax
fetch(dataurl)
    .then(function(resp) {
    return resp.json();
})
.then(function(data) {
    L.geoJson(data, {
         onEachFeature: function onEachFeature(feature, layer) {
            var props = feature.properties;
            var content = `<h3>${props.name}</h3>`;
            layer.bindPopup(content);
      }}).addTo(map);
});        

```

No need to mess with the Django stuff as we should already have a geojson feed.  Looking at our new homepage, we have some leaflet happening:   

![_config.yml]({{ site.baseurl }}/images/post4/leaflet_basic.png)

The points I put in the database are also showing up, and have popups bound to the mouse click.  Note that there isn't really much functional difference between the OpenLayers map and the Leaflet version other than some styling and I believe either library is a good choice.   
