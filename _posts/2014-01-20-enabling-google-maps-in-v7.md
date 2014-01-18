---
layout: post
title: Enabling Google Maps datatype in Umbraco v7
description: A few blog posts ago, I promised to show you how easy it is to enable Google Maps datatype support in V7. Here's how.
category: articles
tags: [umbraco, v7, propertyeditors, google, maps]
---

No Google Maps datatype support without first installing Umbraco v7 of course. If you haven't done so, it now time to perform a Umbraco v7 installation. And use whatever you prefer. Download either from [Our, current release is v7.0.1](http://our.umbraco.org/contribute/releases/701) and use WebMatrix to open up your local copy or start a new Visual Studio.net project and get Umbraco through Nuget. I've also installed a starter kit to have some content already in place.

Once you've got that covered, have a look at the <code>~/Umbraco/Views/PropertyEditors</code> folder of your installation. 

'[PropertyEditors folder in ~/Umbraco/Views](/images/posts/googe-maps-folder-in-property-editors-folder.png)

Also noticed the <code>googlemaps</code> folder? Does that mean we get "Google Maps" property editor support out of the box. No! And if you get it working, it won't exactly be a "Google Maps" datatype you know from similar Google Maps datatype packages you may find on the [interweb](http://our.umbraco.org/search?q=Google%20maps&content=project,). Even worse, it's not actually built to be a fully fledged "Google Maps" property editor. It's been more of a proof of concept and been shown on several Umbraco festivals to give a nice intro on the endless possibilities/opportunities in working with AngularJs.

But don't worry, we'll shamelessly use Per's sample code to create our own simple "Google Maps" property editor.

Hey, wait, there's only one html file in the <code>~/Umbraco/Views/PropertyEditors/googlemaps</code> folder. 

{% highlight html linenos %}
{% raw %}
<div ng-controller="Umbraco.PropertyEditors.GoogleMapsController" class="umb-editor umb-googlemaps">
    <div class="" style="height: 400px;" id="{{model.alias}}_map"></div>
</div>
{% endraw %}
{% endhighlight %}

And where do I find the <code>Umbraco.PropertyEditors.GoogleMapsController</code>?

Well, Umbraco centralizes all controller javascript into a single file, which can be found in <code>~/Umbraco/Js folder</code> (<code>umbraco.controller.js</code>). Below is a snippet we need to enable "Google Maps" support.

{% highlight javascript linenos %}
{% raw %}
angular.module("umbraco")
.controller("Umbraco.PropertyEditors.GoogleMapsController",
    function ($rootScope, $scope, notificationsService, dialogService, assetsService, $log, $timeout) {

        assetsService.loadJs('http://www.google.com/jsapi')
            .then(function () {
                google.load("maps", "3",
                            {
                                callback: initMap,
                                other_params: "sensor=false"
                            });
            });

        function initMap() {
            //Google maps is available and all components are ready to use.
            var valueArray = $scope.model.value.split(',');
            var latLng = new google.maps.LatLng(valueArray[0], valueArray[1]);
            var mapDiv = document.getElementById($scope.model.alias + '_map');
            var mapOptions = {
                zoom: $scope.model.config.zoom,
                center: latLng,
                mapTypeId: google.maps.MapTypeId[$scope.model.config.mapType]
            };
            var geocoder = new google.maps.Geocoder();
            var map = new google.maps.Map(mapDiv, mapOptions);

            var marker = new google.maps.Marker({
                map: map,
                position: latLng,
                draggable: true
            });

            google.maps.event.addListener(map, 'click', function (event) {

                dialogService.mediaPicker({
                    scope: $scope, callback: function (data) {
                        var image = data.selection[0].src;

                        var latLng = event.latLng;
                        var marker = new google.maps.Marker({
                            map: map,
                            icon: image,
                            position: latLng,
                            draggable: true
                        });

                        google.maps.event.addListener(marker, "dragend", function (e) {
                            var newLat = marker.getPosition().lat();
                            var newLng = marker.getPosition().lng();

                            codeLatLng(marker.getPosition(), geocoder);

                            //set the model value
                            $scope.model.vvalue = newLat + "," + newLng;
                        });

                    }
                });
            });

            $('a[data-toggle="tab"]').on('shown', function (e) {
                google.maps.event.trigger(map, 'resize');
            });
        }

        function codeLatLng(latLng, geocoder) {
            geocoder.geocode({ 'latLng': latLng },
                function (results, status) {
                    if (status == google.maps.GeocoderStatus.OK) {
                        var location = results[0].formatted_address;
                        $rootScope.$apply(function () {
                            notificationsService.success("Peter just went to: ", location);
                        });
                    }
                });
        }

        //here we declare a special method which will be called whenever the value has changed from the server
        //this is instead of doing a watch on the model.value = faster
        $scope.model.onValueChanged = function (newVal, oldVal) {
            //update the display val again if it has changed from the server
            initMap();
        };
    });
{% endraw %}
{% endhighlight %}

So, we've got our html and javascript file, why don't we have a Google Maps property editor available when creating a new datatype? Well, there's probably good reasons for that, and pretty sure the fact that this is just sample code is one of those reasons. I guess it's up to the community to create a property editor that actually works ;)

##Enable Google Maps property editor, take 1

All it takes to enable the property editor, is create a <code>package.manifest</code> and put this file in <code>~/App_Plugins/GMaps</code> folder and reference the files <code>umbraco.controller.js</code> and <code>googlemaps.html</code> that are already in place.
 
{% highlight javascript linenos %}
{% raw %}
{
	propertyEditors:
	[ 
		{
			alias: "Netaddicts.Gmaps",
			name: "Google maps",
			editor:
				{
        			view: "~/Umbraco/Views/propertyeditors/googlemaps/googlemaps.html"
        		}
        }
	],
	javascript:
	[
		"~/Umbraco/Js/umbraco.controller.js"
	]
}
{% endraw %}
{% endhighlight %}

Restart your application so Umbraco can pick up your new <code>package.manifest</code> file and you should be able to see a "Google Maps" entry in the dropdown list when creating a new datatype from the "Developer" section. Add your newly created datatype on a new or existing document type and watch the magic. Hell wait, it doesn't work? I told you it's just proof of concept code and needs polishing.

There's a few reasons why it doesn't work yet:

* We don't have a model value yet ($scope.model.value), so next javascript statements will yield invalid code and throw errors

{% highlight javascript linenos %}
{% raw %}
var valueArray = $scope.model.value.split(',');
var latLng = new google.maps.LatLng(valueArray[0], valueArray[1]);
{% endraw %}
{% endhighlight %}

* We don't have any configuration ($scope.model.config), so next javascript statements will fail as well

{% highlight javascript linenos %}
{% raw %}
var mapOptions = {
	zoom: $scope.model.config.zoom,
	center: latLng,
	mapTypeId: google.maps.MapTypeId[$scope.model.config.mapType]
};
{% endraw %}
{% endhighlight %}

##Enable Google Maps property editor, take 2

Forget about what we've done before, we'll take another approach. Forget about configuration options as well, we'll start with a simple property editor that allows for selecting a location on a map and saving that lat-long combination in the database.

Let's change our <code>package.manifest</code> file:

{% highlight javascript linenos %}
{% raw %}
{
	propertyEditors:
	[ 
		{
			alias: "Netaddicts.Gmaps",
			name: "Google maps",
			editor:
				{
        			view: "~/App_Plugins/GMaps/GMaps.html"
        		}
        }
	],
	javascript:
	[
		"~/App_Plugins/GMaps/GMaps.controller.js"
	]
}
{% endraw %}
{% endhighlight %}

We've changed the location of the view and path to our controller which will drive the view.

Make some minor changes to the html view to point to our own custom controller <code>GMaps.GoogleMapsController</code>

{% highlight html linenos %}
{% raw %}
<div ng-controller="GMaps.GoogleMapsController" class="umb-editor umb-googlemaps">
    <div class="" style="height: 400px;" id="{{model.alias}}_map"></div>
</div>
{% endraw %}
{% endhighlight %}

And lastly, our javascript code

{% highlight javascript linenos %}
{% raw %}
angular.module("umbraco").controller("GMaps.GoogleMapsController",
    function ($rootScope, $scope, notificationsService, dialogService, assetsService) {

        assetsService.loadJs('http://www.google.com/jsapi')
            .then(function () {
                google.load("maps", "3",
                            {
                                callback: initMap,
                                other_params: "sensor=false"
                            });
            });

        function initMap() {
            //Google maps is available and all components are ready to use.
            notificationsService.warning("GMaps", "Started initMap()");

            if ($scope.model.value === '') {
                $scope.model.value = '51.226523,2.923182';
            }

            var valueArray = $scope.model.value.split(',');
            var latLng = new google.maps.LatLng(valueArray[0], valueArray[1]);

            //notificationsService.warning("GMaps", "Latitude=" + valueArray[0]);
            //notificationsService.warning("GMaps", "Longitude=" + valueArray[1]);

            var mapDiv = document.getElementById($scope.model.alias + '_map');

            var mapOptions = {
                zoom: 12,
                center: latLng,
                mapTypeId: google.maps.MapTypeId.ROADMAP
            };

            var geocoder = new google.maps.Geocoder();
            var map = new google.maps.Map(mapDiv, mapOptions);

            var marker = new google.maps.Marker({
                map: map,
                position: latLng,
                draggable: true
            });

            //notificationsService.warning("GMaps", "Finished initMap()");

            google.maps.event.addListener(marker, "dragend", function (e) {
                var newLat = marker.getPosition().lat();
                var newLng = marker.getPosition().lng();

                codeLatLng(marker.getPosition(), geocoder);

                //set the model value
                $scope.model.value = newLat + "," + newLng;
            });

            $('a[data-toggle="tab"]').on('shown', function (e) {
                google.maps.event.trigger(map, 'resize');
            });
        }

        function codeLatLng(latLng, geocoder) {
            geocoder.geocode({ 'latLng': latLng },
                function (results, status) {
                    if (status == google.maps.GeocoderStatus.OK) {
                        var location = results[0].formatted_address;
                        $rootScope.$apply(function () {
                            notificationsService.success("Peter just went to: ", location);
                        });
                    } else {
                        notificationsService.error("Peter just went nowhere!");
                    }
                });
        }
    });
{% endraw %}
{% endhighlight %}

Important changes made compared to the original version:

* Checking whether we've got a model value and if not, initialize it with a default lat-long value (Ok, I live in Belgium, so this location will probably be of no use to you, but feel free to change it)

{% highlight javascript linenos %}
{% raw %}
if ($scope.model.value === '') {
	$scope.model.value = '51.226523,2.923182';
}
{% endraw %}
{% endhighlight %}

* Setting map type to <code>google.maps.MapTypeId.ROADMAP</code>

{% highlight javascript linenos %}
{% raw %}
if ($scope.model.value === '') {
	$scope.model.value = '51.226523,2.923182';
}
{% endraw %}
{% endhighlight %}

* Removed some obsolete code (Such as using the dialogService to pick a media item, we won't be using it here) 

Restart your application (this is really, really important for changes to picked up again!) and optionally clear your cache (And you may do so a few times before changes actually show up). Enter the backoffice and you should now have your "Google Maps" datatype working!

<iframe width="420" height="315" src="http://screencast.com/t/JOcOttjQI" frameborder="0" allowfullscreen></iframe>

Big thanks to Per for writing the initial code, I just did what needed be done to make it act as a working Google Maps property editor.

Can we still improve? Yes!!

* How about adding configuration options to avoid hard coding a default location. [Requires some changes to your manifest file](http://umbraco.github.io/Belle/#/tutorials/manifest). Or, and this would make it really awesome, let user pick a location using a "Google Maps Prevalue Editor". Sounds like a new challenge ;)
* Add input box to allow for entering an address and have Google perform a lookup of location
* Allow for multiple locations to be selected on a single map

Happy property editing!