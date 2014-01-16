---
layout: post
title: A developer's take on sending emails in Umbraco
description: As an Umbraco developer spending most (if not all) time in vs.net, I prefer to use ActionMailer. I'll show how easy it is to integrate this Nuget package into your new and existing Umbraco solutions.
category: articles
tags: [umbraco, nuget, actionmailer, vsnet]
---

Being triggered by a tweet of [David Brendel](https://twitter.com/byte5db/status/423077560180678656), asking whether it would be possible to send a rendered view as body of an email in Umbraco, I thought I share my solution on the subject.

Simple answer: Yes, it can be done, and it won't take much c# code, I promise!

Start by installing package [ActionMailer](http://www.nuget.org/packages/ActionMailer/) using Nuget. Next, add a new controller <code>MailController</code> in your project. Make sure it inherits from <code>MailerBase</code>

{% highlight c# linenos %}
{% raw %}
using ActionMailer.Net.Mvc;

namespace Controllers
{
    public class MailController : MailerBase { }
}
{% endraw %}
{% endhighlight %}

Add a new controller Action 

{% highlight c# linenos %}
{% raw %}
public EmailResult ResetPasswordRequest(string newPassword)
{
    From = string.Format("{0} <{1}>", "Sender's name", "sender@domain.com");
    To.Add("recipient@domain.com");
    Subject = "Reset password request";

    return Email("ResetPasswordRequest", newPassword);
}
{% endraw %}
{% endhighlight %}

You're almost done. We need a view, right? Add a view ResetPasswordRequest.html.cshtml in the <code>~/Views/Mail</code> folder (Yes, it follows mvc view location conventions). It's important to add the <code>.html</code> suffix to the name of your view you're referencing in your controller's action (ie <code>ResetPasswordRequest</code>), as <code>ActionMailer</code> requires so (And it does so because it allows to send multipart messaging, meaning it can render views in html, text, or both - if you wish to add text support, just add another view with the same name and <code>.txt</code> suffix in the same view folder location)

{% highlight html linenos %}
{% raw %}
@model string

@{
    Layout = "Master.html.cshtml";
}
<p>Hiya, you've requested a new password. Here it is: @Model</p>
{% endraw %}
{% endhighlight %}

Of course you can pass in whatever type as model of your view, it's just mvc! Also, above code snippet assumes you've got a master layout already in place.

Now you're ready to send your first email using ActionMailer. From any controller or view, just use

{% highlight c# linenos %}
{% raw %}
new MailController().ResetPasswordRequest("AFancyNewPassword").Deliver();
{% endraw %}
{% endhighlight %}

That's it! I told you it wouldn't take much effort! 

Don't forget to add smtp configuration in <code>web.config</code>, otherwise no emails will be sent. While in development, it's a good idea to use a folder on your hard disk to save emails to. All mails will be available as .eml files which can be opened in any mail application for inspection. Also make sure application pool identity has write permissions on the specified folder!

{% highlight xml linenos %}
{% raw %}
<system.net>
  <mailSettings>
    <smtp deliveryMethod="SpecifiedPickupDirectory">
      <specifiedPickupDirectory pickupDirectoryLocation="C:\Inetpub\Mail"/>
    </smtp>
  </mailSettings>
</system.net>
{% endraw %}
{% endhighlight %}

Are there any caveats? Yes unfortunately...

* Your email views MUST go into its mvc conventional view folder, making it impossible to change templates/views from within the Umbraco admin interface.

Fortunately, it has never been a showstopper for us. Most of the times, a mail template doesn't change ever, or maybe a few times before you actually go live.

Happy mailing!