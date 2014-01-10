---
layout: post
title: Umbraco v7 XPathDropdownList/Google Maps support
description: "A quick intro on how to build a simplified version of the XPathDropdownList and enable the Google Maps datatype in Umbraco v7"
category: articles
tags: [umbraco, v7, propertyeditors]
---

[Umbraco v7](http://our.umbraco.org/contribute/releases/701) comes with a bunch of property editors available out of the box. All these property editors are available in the dropdown when creating a new datatype. 

Last week, I started porting a site from v6 to v7 as I was eager to find out how it would look like in the new version. Also, that v6 site was a perfect candidate as the site wasn't too complex in terms of content and structure. Most of the stuff could be done with no major modifications.

However, 2 important things were missing in our setup. First one was the XPathDropdownList we used to know from [uComponents](http://ucomponents.org) but have been integrated into the core, 2nd one was a Google maps datatype. For the last one, there's a couple of packages available on [Our](http://our.umbraco.org). But none of these have been ported over to v7 (yet) afaik. 

Sounds like a fun way to learn about building property editors, right? Yup!

##XPathDropdownList

Actually, I didn't need a fully fledged XPathDropdownList datatype, but just needed a datatype that could list all child documents of a specific node in the tree.

