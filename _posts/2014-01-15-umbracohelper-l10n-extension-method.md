---
layout: post
title: UmbracoHelper.L10N()
description: "A useful UmbracoHelper extension method to create dictionary items 'on the fly'"
category: articles
tags: [umbraco, translation, localization, l10n]
---

Let's be honest, creating dictionary items in Umbraco is not the most sexy and interesting job. As a matter of fact, I must admit I just hate creating dictionary items via the Umbraco admin interface. And because I'm dealing with lots of multilingual websites, I had to find a way to make my life a bit easier... so I knocked together a small extension method which creates dictionary item at run time, whenever they're requested. With an option to set a default translation.

As I wanted to use this extension method from any context, whether in a view (or template if you prefer) or a controller... I went ahead and wrote an extension method on the <code>UmbracoHelper</code> object, which is almost always available.

I added a few more functional requirements:
 
* I want to be able to derive usage of a dictionary item based on it's key (eg. Project.Views.TemplateX.KeyY expresses the use of 'KeyX' dictionary item in the context of template 'TemplateY'.
* I want translations to be created for each of the installed languages.
* Only create dictionary items in debug mode (or development mode!).
* Build a hierarchical structure to avoid a looooooong flat list of dictionary items in the backend.

{% highlight c# linenos %}
{% raw %}
using System;
using System.Linq;
using System.Web;
using Umbraco.Core.Models;
using Umbraco.Web;
using log4net;

namespace Project.Business.Extensions
{
    public static class UmbracoHelperExtensions
    {
        private static readonly ILog Logger = LogManager.GetLogger(typeof(UmbracoHelperExtensions));

        public static string L10N(this UmbracoHelper helper, string key, string value)
        {
            var translation = helper.GetDictionaryValue(key);

            if (HttpContext.Current.IsDebuggingEnabled && string.IsNullOrEmpty(translation))
            {
                try
                {
                    var dictionaryService = UmbracoContext.Current.Application.Services.LocalizationService;
                    var tryGetDictionaryItem = dictionaryService.GetDictionaryItemByKey(key);

                    if (tryGetDictionaryItem != null)
                    {
                        return tryGetDictionaryItem.Translations.First(x => x.Language.CultureName == UmbracoContext.Current.PublishedContentRequest.Culture.Name).Value;
                    }

                    var dictionaryItemParts = key.Split(new[] { "." }, StringSplitOptions.RemoveEmptyEntries);

                    var topLevelKey = dictionaryItemParts[0];

                    var languages = dictionaryService.GetAllLanguages().ToList();
                    IDictionaryItem dictionaryItem = null;

                    if (!dictionaryService.DictionaryItemExists(topLevelKey))
                    {
                        dictionaryItem = new DictionaryItem(topLevelKey)
                        {
                            Translations = languages.Select(language => new DictionaryTranslation(language, string.Format("[{0}] {1}", language.CultureName, topLevelKey))).Cast<IDictionaryTranslation>().ToList()
                        };

                        dictionaryService.Save(dictionaryItem);
                    }
                    else
                    {
                        dictionaryItem = dictionaryService.GetDictionaryItemByKey(topLevelKey);
                    }

                    var foreachKey = topLevelKey;
                    var nonTopLevelKeys = dictionaryItemParts.ToList().Skip(1);

                    foreach (var nonTopLevelKey in nonTopLevelKeys)
                    {
                        foreachKey += string.Format(".{0}", nonTopLevelKey);

                        if (!dictionaryService.DictionaryItemExists(foreachKey))
                        {
                            var newDictionaryItem = new DictionaryItem(dictionaryItem.Key, foreachKey)
                            {
                                Translations = languages.Select(language => new DictionaryTranslation(language, string.Format("[{0}] {1}", language.CultureName, key == foreachKey ? value : foreachKey))).Cast<IDictionaryTranslation>().ToList()
                            };

                            dictionaryService.Save(newDictionaryItem);
                            dictionaryItem = newDictionaryItem;
                        }
                        else
                        {
                            dictionaryItem = dictionaryService.GetDictionaryItemByKey(foreachKey);
                        }
                    }
                }
                catch (Exception exception)
                {
                    Logger.Error("Ouch, error translation item in debug mode", exception);
                }
            }

            return helper.GetDictionaryValue(key);
        }
    }
}
{% endraw %}
{% endhighlight %}

Code is pretty self-explanatory, so I won't go into detail on that.

Let's explain the usage of the extension method. Assuming I'm editing a template <code>RegisterPage.cshtml</code> in the <code>~/Views</code> folder.

{% highlight html linenos %}
{% raw %}
<h1>@Umbraco.L10N("Project.Views.RegisterPage.Title", "Registratie")</h1>
{% endraw %}
{% endhighlight %}

It'll only take a single frontend hit of any node in your Umbraco installation associated with this template to create a hierarchical structure of dictionary items.

{% highlight html linenos %}
{% raw %}
Project
	Project.Views
		Project.Views.RegisterPage
			Project.Views.RegisterPage.Title
{% endraw %}
{% endhighlight %}

But at least, I will have created translations for each of the installed languages. So, for example, if I was working on a nl-BE and fr-BE multilingual site, our extension method would have created two translations for our key Project.Views.RegisterPage.Title in both Dutch and French

Just a little catch: Make sure you've got your culture set before you start using the extension method, otherwise your dictionary items won't be created!

![Setting cuture](/images/posts/culture-and-hostnames-l10n.png)

Can we make it better? Yes, we probably can... Here's a list of things which still annoy me:

* Dictionary keys are strings and therefore error prone. Hard to change a key, unless you correct your typo and browse the same page again to create the corresponding dictionary items.
* How about "General" translations? A submit button is supposed to be set to the same text throughout the entire site. My hierarchical structure does allow for that, but you'll lose contextual info because your key won't be named according to it's use.

I'm also very looking forward on how other people are dealing with multilingual sites and dictionary items! So please comment if you know better or other implementations.
