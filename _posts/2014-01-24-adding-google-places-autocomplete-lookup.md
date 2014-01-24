---
layout: post
title: Google Maps datatype in Umbraco v7 - Adding Google Places Autocomplete lookup functionality
description: Following up on my last blog post on enabling Google Maps property editor and driven by community comments received, I created a new version of the property editor with Google Places Autocomplete lookup. 
category: umbraco
tags: [umbraco, v7, propertyeditors, google, maps, places]
---

I received some feedback from Bjarne Fyrstenborg [on my previous blog post](http://www.netaddicts.be/umbraco/enabling-google-maps-in-v7/), suggesting to look into adding a jquery address picker. It was already on my list of possible extensions, so went ahead and checked the references from his comments.

I wasn't completely overwhelmed by its look & feel and went to search for alternatives. Knew about [Google Places](https://developers.google.com/maps/documentation/javascript/places), but funny enough hadn't heard about [Google Places Autocomplete](https://developers.google.com/maps/documentation/javascript/places-autocomplete)!

And because Umbraco v7 is largely driven by AngularJs, I also searched whether there was something already available combining those two. Guess what? I does exist. Found the [ngAutocomplete](http://ngmodules.org/modules/ngAutocomplete) Angular directive to add Google Places Autocomplete to a textbox element. Exactly what I needed for my property editor.

Kind of disappointing to realize the advanced usage sample was built against Angular version 1.2.4. Ok, [took me some time to realize I couldn't use that version in Umbraco as per this thread](https://groups.google.com/forum/#!topic/umbraco-dev/uwDsJZGElfQ). 

Oh, well, let's try to make it work ourselves. Finally, after a few hours of code fiddling, I got myself a first working version... Did some polishing afterwards, ironed out some minor issues and packaged it all up for you to grab and enjoy! 

![Google Maps Property Editor w/ Google Places Autocomplete lookup](/images/posts/google-maps-places-autocomplete.png)

[Get your copy!](http://our.umbraco.org/projects/backoffice-extensions/google-maps-property-editor-w-google-places-autocomplete-lookup)

Feedback and comments much appreciated.

Happy property editing!