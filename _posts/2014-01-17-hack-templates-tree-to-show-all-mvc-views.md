---
layout: post
title: Editing mvc views in the Umbraco backoffice
description: A follow up on my previous blog post on ActionMailer introduces a hacky way to edit mvc views from the Umbraco backoffice.
category: articles
tags: [umbraco, vsnet, backoffice]
---

In my [last post](http://www.netaddicts.be/articles/a-developers-take-on-sending-emails-in-umbraco/), I talked about a single caveat with using "real" mvc views... those can't be edited via the Umbraco backoffice. 
I did some digging to find out whether there's a way to do it anyway. After all, it can't be that hard to recursively enumerate all <code>.cshtml</code> (or <code>.vbhtml</code>) files in the standard mvc views location folder <code>~/Views</code> and have those edited.

For example, suppose you've got a controller <code>SubscriptionSurfaceController</code> with an action <code>SubscriptionForm</code> as in following code snippet

{% highlight c# linenos %}
{% raw %}
using System.Web.Mvc;
using Umbraco.Web.Mvc;

namespace Controllers
{
    public class SubscriptionViewModel { }

    public class SubscriptionSurfaceController : SurfaceController
    {
        public ActionResult SubscriptionForm()
        {
            return View(new SubscriptionViewModel());
        }
    }
}
{% endraw %}
{% endhighlight %}

you'd expect the corresponding view <code>SubscriptionForm.cshtml</code> to live in the <code>~/Views/SubscriptionSurface</code> folder.

But that makes it hard for people that use the admin backoffice to make changes to this view as you can't access that file from the Umbraco admin. Yes, of course you can still put your view wherever you want, as long as you tell the system where your view will reside. It would only take a simple change to the action <code>SubscriptionForm</code> returning the view

{% highlight c# linenos %}
{% raw %}
return View("~/Views/FolderA/SubfolderB/MyView.cshtml", new SubscriptionViewModel());
{% endraw %}
{% endhighlight %}

And to make it even more Umbraco-ish, put your view in the default Umbraco views folder <code>~/Views</code>

{% highlight c# linenos %}
{% raw %}
return View("~/Views/MyView.cshtml", new SubscriptionViewModel());
{% endraw %}
{% endhighlight %}

Ok, so why didn't you put all of your required mailing views ([remember, from my last post on ActionMailer](http://www.netaddicts.be/articles/a-developers-take-on-sending-emails-in-umbraco/)) in this folder using the same construction? Well, it's called legacy... I can't change the way <code>ActionMailer</code> is designed to work, and I don't want to, because it feels more mvc-ish. And I like that. I had to find a way to still be able to edit those...

It turns out to be a bit of a hack... But hey, it works! I started out by making a copy of the <code>loadTemplates</code> class which is responsible for listing all templates in your Umbraco installation, and put that copy in my Umbraco project. Renamed <code>loadTemplates</code> into <code>LoadViews</code> and made a few changes to the original code.

{% highlight c# linenos %}
{% raw %}
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Umbraco.Core;
using Umbraco.Core.IO;
using umbraco;
using umbraco.BusinessLogic.Actions;
using umbraco.businesslogic;
using umbraco.cms.businesslogic.template;
using umbraco.cms.presentation.Trees;
using umbraco.interfaces;

namespace Project.Business
{
    [Tree(Umbraco.Core.Constants.Applications.Settings, "views", "Views", sortOrder: 11)]
    public class LoadProjectViews : BaseTree
    {
        public LoadProjectViews(string application) : base(application) { }

        protected override void CreateRootNode(ref XmlTreeNode rootNode)
        {
            rootNode.NodeType = "init" + TreeAlias;
            rootNode.NodeID = "init";
        }

        public override void RenderJS(ref StringBuilder Javascript)
        {

            Javascript.Append(
                @"
                function openTemplate(id) {
                        UmbClientMgr.contentFrame('settings/editTemplate.aspx?templateID=' + id);
                }   
                
                 function openView(id) {
                    UmbClientMgr.contentFrame('settings/views/editView.aspx?treeType=templates&templateID=' + id);
                            }

                function openSkin(id) {
                        UmbClientMgr.contentFrame('settings/editSkin.aspx?skinID=' + id);
                }
                ");
        }


        public override void Render(ref XmlTree tree)
        {
            string folder = umbraco.library.Request("folder");
            string folderPath = umbraco.library.Request("folderPath");

            if (!string.IsNullOrEmpty(folder))
                RenderTemplateFolderItems(folder, folderPath, ref tree);
            else
            {
                if (UmbracoSettings.EnableTemplateFolders)
                    RenderTemplateFolders(ref tree);

                RenderTemplates(ref tree);
            }
        }

        private void RenderTemplateFolderItems(string folder, string folderPath, ref XmlTree tree)
        {
            string relPath = SystemDirectories.Masterpages + "/" + folder;
            if (!string.IsNullOrEmpty(folderPath))
                relPath += folderPath;

            string fullPath = IOHelper.MapPath(relPath);

            foreach (string dir in System.IO.Directory.GetDirectories(fullPath))
            {
                System.IO.DirectoryInfo directoryInfo = new DirectoryInfo(dir);
                XmlTreeNode xNode = XmlTreeNode.Create(this);
                xNode.Menu.Clear();
                xNode.Menu.Add(ActionRefresh.Instance);
                xNode.NodeID = "-1";
                xNode.Text = directoryInfo.Name;
                xNode.HasChildren = true;
                xNode.Icon = "folder.gif";
                xNode.OpenIcon = "folder_o.gif";
                xNode.Source = GetTreeServiceUrl(directoryInfo.Name) + "&folder=" + folder + "&folderPath=" + folderPath + "/" + directoryInfo.Name;
                tree.Add(xNode);
            }

            foreach (string file in System.IO.Directory.GetFiles(fullPath))
            {
                System.IO.FileInfo fileinfo = new FileInfo(file);
                string ext = fileinfo.Extension.ToLower().Trim('.');

                XmlTreeNode xNode = XmlTreeNode.Create(this);
                xNode.Menu.Clear();
                xNode.Menu.Add(ActionRefresh.Instance);
                xNode.NodeID = "-1";
                xNode.Text = Path.GetFileName(file);
                xNode.HasChildren = false;
                xNode.Action = "javascript:openScriptEditor('" + relPath + "/" + Path.GetFileName(file) + "');";

                //tree.Add(xNode);


                switch (ext)
                {
                    case "master":
                        xNode.Icon = "settingTemplate.gif";
                        xNode.OpenIcon = "settingTemplate.gif";
                        tree.Add(xNode);
                        break;
                    case "css":
                    case "js":
                        xNode.Icon = "settingsScript.gif";
                        xNode.OpenIcon = "settingsScript.gif";
                        tree.Add(xNode);
                        break;
                    case "xml":
                        if (xNode.Text == "skin.xml")
                        {
                            xNode.Icon = "settingXml.gif";
                            xNode.OpenIcon = "settingXml.gif";
                            tree.Add(xNode);
                        }
                        break;
                    case "cshtml":
                        xNode.Icon = "settingView.gif";
                        xNode.OpenIcon = "settingView.gif";
                        tree.Add(xNode);
                        break;
                    default:
                        break;
                }



                //xNode.Source = GetTreeServiceUrl(s.Alias) + "&skin=" + skin + "&path=" + path;
            }


        }

        private void RenderTemplateFolders(ref XmlTree tree)
        {
            if (base.m_id == -1)
            {
                foreach (string s in Directory.GetDirectories(IOHelper.MapPath(SystemDirectories.Masterpages)))
                {
                    var _s = Path.GetFileNameWithoutExtension(s);

                    XmlTreeNode xNode = XmlTreeNode.Create(this);
                    xNode.NodeID = _s;
                    xNode.Text = _s;
                    xNode.Icon = "folder.gif";
                    xNode.OpenIcon = "folder_o.gif";
                    xNode.Source = GetTreeServiceUrl(_s) + "&folder=" + _s;
                    xNode.HasChildren = true;
                    xNode.Menu.Clear();
                    xNode.Menu.Add(ActionRefresh.Instance);

                    OnBeforeNodeRender(ref tree, ref xNode, EventArgs.Empty);
                    if (xNode != null)
                    {
                        tree.Add(xNode);
                        OnAfterNodeRender(ref tree, ref xNode, EventArgs.Empty);
                    }
                }
            }
        }

        private void RenderTemplates(ref XmlTree tree)
        {
            List<Template> templates = null;
            if (base.m_id == -1)
                templates = Template.GetAllAsList().FindAll(delegate(Template t) { return !t.HasMasterTemplate; });
            else
                templates = Template.GetAllAsList().FindAll(delegate(Template t) { return t.MasterTemplate == base.m_id; });

            foreach (Template t in templates)
            {
                XmlTreeNode xNode = XmlTreeNode.Create(this);
                xNode.NodeID = t.Id.ToString();
                xNode.Text = t.Text;
                xNode.Source = GetTreeServiceUrl(t.Id);
                xNode.HasChildren = t.HasChildren;

                //if (Umbraco.Core.Configuration.UmbracoSettings.DefaultRenderingEngine == RenderingEngine.Mvc && ViewHelper.ViewExists(t))
                //{
                    xNode.Action = "javascript:openView(" + t.Id + ");";
                    xNode.Icon = "settingView.gif";
                    xNode.OpenIcon = "settingView.gif";
                //}
                //else
                //{
                //    xNode.Action = "javascript:openTemplate(" + t.Id + ");";
                //    xNode.Icon = "settingTemplate.gif";
                //    xNode.OpenIcon = "settingTemplate.gif";
                //}

                if (t.HasChildren)
                {
                    xNode.Icon = "settingMasterTemplate.gif";
                    xNode.OpenIcon = "settingMasterTemplate.gif";
                }

                OnBeforeNodeRender(ref tree, ref xNode, EventArgs.Empty);

                if (xNode != null)
                {
                    tree.Add(xNode);
                    OnAfterNodeRender(ref tree, ref xNode, EventArgs.Empty);
                }

            }

        }

        protected override void CreateAllowedActions(ref List<IAction> actions)
        {
            actions.Clear();
            actions.AddRange(new IAction[] { ActionNew.Instance, ActionDelete.Instance, 
                ContextMenuSeperator.Instance, ActionRefresh.Instance });
        }
    }
}
{% endraw %}
{% endhighlight %}

Compared to the original <code>loadTemplates</code> class, I've:

* Made some changes to the <code>Tree</code> attribute settings to make sure it wouldn't collide with an existing alias. We still want the original templates to be available under 'Templates'
* Added a new case <code>".cshtml"</code> in <code>RenderTemplateFolderItems()</code> function
* Put some code in comments in function <code>RenderTemplates()</code> as otherwise it wouldn't compile because of the use of internal functions. Again, not a biggie for me as I'm always in Mvc rendering mode

And then I made some configuration changes to both <code>umbracoSettings.config</code> and <code>web.config</code>.

Changes in <code>umbracoSettings.config</code>

* Add <code>/views</code> folder in <code>scriptFolderPath</code> tag
* Add <code>cshtml</code> file type in <code>scriptFileTypes</code> tag
* Add extra tag <code>enableTemplateFolders</code> with value <code>true</code> under <code>templates</code> tag

{% highlight xml linenos %}
{% raw %}
<scripteditor>
  <!-- Path to script folder - no ending "/" -->
  <scriptFolderPath>/scripts,/views</scriptFolderPath>
  <!-- what files can be opened/created in the script editor -->
  <scriptFileTypes>js,xml,cshtml</scriptFileTypes>
  <!-- disable the codemirror editor and use a simple textarea instead -->
  <scriptDisableEditor>false</scriptDisableEditor>
</scripteditor>
...
<templates>
  <!-- If you want to switch to Mvc then do NOT change useAspNetMasterPages to false -->
  <!-- This (old!) setting is still used to control how macros are inserted into your pages -->
  <useAspNetMasterPages>true</useAspNetMasterPages>
  <!-- To switch the default rendering engine to MVC, change this value from WebForms to Mvc -->
  <!-- Do not set useAspNetMasterPages to false, it is not relevant to MVC usage -->
  <defaultRenderingEngine>Mvc</defaultRenderingEngine>
  <enableTemplateFolders>true</enableTemplateFolders>
</templates>
{% endraw %}
{% endhighlight %}

Changes in <code>web.config</code>:

* Add a new application settings <code>umbracoMasterPagesPath</code> with value <code>~/Views</code>

{% highlight xml linenos %}
{% raw %}
<appSettings>
  <add key="umbracoMasterPagesPath" value="~/Views"/> 
</appSettings>
{% endraw %}
{% endhighlight %}

Build your Umbraco project and/or restart your application and you should now see an extra tree node <code>Views</code> in <code>Settings</code> section.

![Editing your mvc views](/images/posts/hacking-mvc-views-folder.png)

Btw, this has just been tested on a v6.1.3 Umbraco installation! I guess it will still be valid on any v6.1+ Umbraco installation.  Of course, this WON'T WORK on v7!! Don't tell me I didn't warn you.

Happy hacking!