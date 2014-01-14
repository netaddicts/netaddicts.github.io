---
layout: post
title: Umbraco v7 XPathDropdownList aka Docr
description: "A quick intro on how to build a VERY simplified version of the XPathDropdownList"
category: articles
tags: [umbraco, v7, propertyeditors]
---

[Umbraco v7](http://our.umbraco.org/contribute/releases/701) comes with a bunch of property editors available out of the box. All these property editors are available in the dropdown when creating a new datatype. 

Last week, I started porting a site from v6 to v7 as I was eager to find out how it would look like in the new version. Also, that v6 site was a perfect candidate as the site wasn't too complex in terms of content and structure. Most of the stuff could be done with no major modifications.

However, 2 important things were missing in our setup. First one was the XPathDropdownList we used to know from [uComponents](http://ucomponents.org) but have been integrated into the core, 2nd one was a Google maps datatype. For the last one, there's a couple of packages available on [Our](http://our.umbraco.org). But none of these have been ported over to v7 (yet) afaik. I'll talk about enabling Google maps support in a later blog post.

Sounds like a fun way to learn about building property editors, right? Yup!

For this sample to work, I'm assuming you've got at least a v7 Umbraco installation with any of a starter kit selected upon installation. If you don't have a starter kit installed, make sure to replace <code>"umbNewsItem"</code> document type alias with any of the available document types in your local installation.

##XPathDropdownList aka Docr

Actually, I didn't need a fully fledged XPathDropdownList datatype, but just needed a datatype that could list all child documents of a specific node in the tree. Here's my initial <code>package.manifest</code> file which I've put in <code>~/App_Plugins/Netaddicts</code> folder

{% highlight html linenos %}
{% raw %}
{
	propertyEditors:
	[ 
		{
			alias: "Netaddicts.Docr",
			name: "Netaddicts Document Selection",
			editor:
			{
				view: "~/App_Plugins/Netaddicts/Docr/Docr.html"
			}
		}
	],
	javascript:
	[
		"~/App_Plugins/Netaddicts/Docr/Docr.controller.js",
		"~/App_Plugins/Netaddicts/Docr/Docr.resources.js"
	]
}
{% endraw %}
{% endhighlight %}

Don't put your manifest file in a subfolder of <code>~/App_Plugins/Netaddicts/</code> (eg. <code>~/App_Plugins/Netaddicts/Docr</code>) which I did initially, but was having troubles having Umbraco pick up this manifest file on application restart. Ironically, if I would have read the documentation on [package manifest file](http://umbraco.github.io/Belle/#/tutorials/manifest), I wouldn't have been bitten by this.

So, now I just need three more files...

Our first file <code>Docr.html</code> will be our view which should just be a dropdown list holding all child nodes of a specific node.

{% highlight html linenos %}
{% raw %}
<div ng-controller="Netaddicts.DocrController">
    <select name="DocrDropdownList" class="umb-editor umb-dropdown" ng-model="model.value" ng-options="d.Id as d.Name for d in documents" />
</div>
{% endraw %}
{% endhighlight %}

Javascript file <code>Docr.controller.js</code> will be responsible to initializing dropdown list with documents fetched from Umbraco content tree. Syntax of the Angular ngOptions directive may seem a bit strange, and at first hard to understand. But there's [good documentation on this](http://docs.angularjs.org/api/ng.directive:select) as well.

{% highlight html linenos %}
{% raw %}
angular.module("umbraco").controller("Netaddicts.DocrController", function ($scope, docrResource, notificationsService) {
	
	docrResource.getAll("umbNewsItem").then(function (response) {
		$scope.documents = response.data;
		//console.log(response.data);
	}, function(response) {
		notificationsService.error("Error", "Error loading documents");
		console.log(response.data);
	});
 }
);
{% endraw %}
{% endhighlight %}

Don't forget to replace umbNewsItem with another document type alias if you've not installed a starter kit!!

Also notice how we inject <code>docrResource</code> and <code>notificationService</code> into our controller. AngularJs uses a technique called [Dependency Injection](http://docs.angularjs.org/guide/di) and uses an [injector](http://docs.angularjs.org/api/angular.injector) to manage the responsibilities of dependency creation.
Don't worry if this sounds too complicated, just remember that whenever you need to use a service or resource, you can just include those in your function argument list when declaring your controller.

In our previous code snippet above, we're injecting Umbraco's <code>notificationsService</code>, used to alert if something goes wrong when fetching data from the Umbraco system using our own custom resource <code>docrResource</code>.

Let's take a look at our <code>Docr.resource.js</code> file.

{% highlight html linenos %}
{% raw %}
angular.module('umbraco.resources').factory('docrResource', function ($q, $http) {
	return {
		getAll: function (documentTypeAlias) {
			return $http.get("Netaddicts/DocrApi/GetDocuments/?documentTypeAlias=" + documentTypeAlias);
		}
	};
});
{% endraw %}
{% endhighlight %}

Above code snippet adds the docrResource to the <code>umbraco.resources</code> module and uses the standard AngularJs factory pattern, so we can inject this resource into any of our controllers.

Next up: A bit of C# coding to actually fetch the documents from Umbraco. Either create a simple C# class file and drop it into the <code>~/App_Code</code> folder or add your class to an existing project and compile into assembly.

{% highlight html linenos %}
{% raw %}
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using Umbraco.Web.Editors;
using Umbraco.Web.Mvc;

namespace Web.Controllers.Api
{
    [PluginController("Netaddicts")]
    public class DocrApiController : UmbracoAuthorizedJsonController
    {
        public IEnumerable<DocumentInfo> GetDocuments(string documentTypeAlias)
        {
            var contentService = ApplicationContext.Services.ContentService;
            var contentTypeService = ApplicationContext.Services.ContentTypeService;

            return contentService.GetContentOfContentType(contentTypeService.GetContentType(documentTypeAlias).Id)
                .Select(x => new DocumentInfo {Id = x.Id.ToString(CultureInfo.InvariantCulture), Name = x.Name});
        }
    }

    public class DocumentInfo
    {
        public string Id { get; set; }
        public string Name { get; set; }
    }
}
{% endraw %}
{% endhighlight %}

Don't forget to add the <code>PluginController</code> attribute so Umbraco can match mvc route <code>/Netaddicts/DocrApi/GetDocuments</code> to your controller action <code>GetDocuments()</code>.
Also notice we're returning a IEnumerable<DocumentInfo> instead of returning an object of type IEnumerable<IContent> just because you can't serialize an interface! Period!

If all files are in place, all you have to do next is go to the Developer section in your umbraco installation, create a new datatype and select "Netaddicts Document Selection" from the list of available types, and add this new datatype to an already existing document type.

Of course, this wasn't yet exactly what I had in mind, but this sample code got me started and made sure all components were in place. All I had to do next was actually fetching data from a specific node in the content tree instead of fetching documents of a specific document type. Enter "Prevalue Editor".

It's always a good idea to take a look at what's already existing. After all, all I wanted was an option to set a "root" node, and only list child nodes of this specific node. And Umbraco comes with quite a few prevalue editors out of the box. For a complete list of prevalue editors, browse the <code>/Umbraco/Views/Prevalueeditors</code> folder in your installation. I will be using the <code>"treepicker"</code> prevalue editor to allow me to select a "root" node.

A small change to our <code>package.manifest</code> file is required...

{% highlight html linenos %}
{% raw %}
{
	propertyEditors:
	[
		{
			alias: "Netaddicts.Docr",
			name: "Netaddicts Document Selection",
			editor:
			{
				view: "~/App_Plugins/Netaddicts/Docr/Docr.html"
			},
			prevalues: 
			{
				fields: 
				[
					{
						label: "Root content",
						description: "Select root content node",
						key: "rootNode",
						view: "treepicker"
					}
				]
			}
		} 
	],
	javascript: 
	[
		"~/App_Plugins/Netaddicts/Docr/Docr.controller.js",
		"~/App_Plugins/Netaddicts/Docr/Docr.resources.js",
	]
}
{% endraw %}
{% endhighlight %}

Add a new controller action to fetch the data

{% highlight html linenos %}
{% raw %}
public IEnumerable<DocumentInfo> GetChildDocuments(int rootNode)
{
	var contentService = ApplicationContext.Services.ContentService;

	return contentService.GetChildren(rootNode)
		.Select(x => new DocumentInfo { Id = x.Id.ToString(CultureInfo.InvariantCulture), Name = x.Name });
}
{% endraw %}
{% endhighlight %}

Make a minor modification to our resource javascript file

{% highlight html linenos %}
{% raw %}
angular.module('umbraco.resources').factory('docrResource', function ($q, $http) {
	return {
		getAll: function (documentTypeAlias) {
			return $http.get("Netaddicts/DocrApi/GetDocuments/?documentTypeAlias=" + documentTypeAlias);
		},
		getChildren: function(rootNode) {
			return $http.get("Netaddicts/DocrApi/GetChildDocuments/?rootNode=" + rootNode);
		}
	};
});
{% endraw %}
{% endhighlight %}

And, lastly, change our controller javascript file

{% highlight html linenos %}
{% raw %}
angular.module("umbraco").controller("Netaddicts.DocrController", function ($scope, docrResource, notificationsService) {

	docrResource.getChildren($scope.model.config.rootNode).then(function (response) {
		$scope.documents = response.data;
		//console.log(response.data);
	}, function () {
		notificationsService.error("Error", "Error loading documents");
		//console.log(response.data);
	});

	docrResource.getAll("umbNewsItem").then(function (response) {
		$scope.documents = response.data;
		//console.log(response.data);
	}, function(response) {
		notificationsService.error("Error", "Error loading documents");
		//console.log(response.data);
	});
});
{% endraw %}
{% endhighlight %}

Notice how we're using the selected "root" node <code>$scope.model.config.rootNode</code> to fetch data using the <code>getChildren()</code> function. <code>"rootNode"</code> is the key we've set in the manifest file.
You should now be able to create multiple datatypes based on the same property editor, each fetching child nodes from a different root node.

Happy property editing.