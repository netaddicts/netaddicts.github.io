---
layout: post
title: Google Maps datatype in Umbraco v7 - Adding a property editor value converter to return a strongly typed object
description: Following up on my last blog post on enabling Google Maps property editor and driven by community comments received (Yes, looking at you [Warren](https://twitter.com/warrenbuckley/status/426735250606391296)), I added support for returning a strongly typed object from my property editor for use in Razor views
category: umbraco
tags: [umbraco, v7, propertyeditors, google, maps, places, editorvalueconverter]
---

Adding support for returning strongly types objects is dead simple. Just make sure to create a class that inherits from <code>PropertyValueConverterBase</code> and override the required functions.

{% highlight c# linenos %}
{% raw %}
public class GMapsValueConverter : PropertyValueConverterBase
{
    public override bool IsConverter(PublishedPropertyType propertyType)
    {
        return "Netaddicts.GMaps".Equals(propertyType.PropertyEditorAlias);
    }

    public override object ConvertDataToSource(PublishedPropertyType propertyType, object source, bool preview)
    {
        if (source == null || string.IsNullOrWhiteSpace(source.ToString())) return null;
     
        var coordinates = source.ToString().Split(new[] {","}, StringSplitOptions.RemoveEmptyEntries);

        return new GMapsLocation
            {
                Lat = decimal.Parse(coordinates[0]),
                Lng = decimal.Parse(coordinates[1])
            };
    }
}
{% endraw %}
{% endhighlight %}

For my <code>Google Maps</code> property editor, I return a <code>GMapsLocation</code> object based on a string stored in the database. As I'm storing the location in <code>GMaps</code> property editor as a comma separated string value, I just need to split that string into two parts and return the latitude/longitude coordinates.

And in my Razor views, I can simply use

{% highlight c# linenos %}
{% raw %}
var gmapLocation = Model.Content.GetPropertyValue<GMapsLocation>("map");
{% endraw %}
{% endhighlight %}

to convert my property value into a strongly types <code>GMapsLocation</code> object.

![Use of editor value converter in Razor views](/images/posts/use-of-value-converter-in-razor-views.png)

End here's the output on the homepage of starterkit installed on a clean v7.0.2 installation.

![Output of use of editor value converter in Razor views](/images/posts/use-of-value-converter-in-razor-views-output.png)

I've updated source code on GitHub. Either get [your copy from there](https://github.com/netaddicts/umbraco-v7-property-editors-gmaps) or download package from [Our](http://our.umbraco.org//projects/backoffice-extensions/google-maps-property-editor-w-google-places-autocomplete-lookup)

Happy property editor value converting!