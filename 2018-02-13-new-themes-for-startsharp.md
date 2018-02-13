---
title: 6 New Themes for StartSharp
description: We think the AdminLTE theme we use in Serene is a beatiful one. But as any hit song you loved last year, gets a bit boring by time, themes also start to look old. So, we have designed 6 new themes for StartSharp.
author: Volkan Ceylan
---

## Looking at Our Options

Before trying to design our own themes, we evaluated other options like integrating one or more popular commercial themes like Metronic, Limitless, Remark etc.

First problem is licencing. As our customers should be able to create and release multiple applications to unlimited users, even if we bought the Extended Licence ( which has a much higher price tag than a regular licence) it wouldn't be legally enough for all StartSharp users to distribute their applications as they wish (e.g. on-premise or SaS applications).

We could make an agreement with authors, but in that case we could only integrate one theme as costs for multiple themes would be too high.

Another problem is maintainability and updates. We had some prior experience with some of these themes, and from version to version they introduce far too many breaking changes. They don't worry about backward compability much as they probably think that you create a site with one version, and when you use the next version, you'll be creating another web site.

It would take some time for us to integrate their changes but as all ours users who created a StartSharp application with a prior version would have to do the same, that's not so acceptable.

But the most critical issue is something else. Even though they are usually based on Bootstrap, all these themes require some kind of specialized HTML markup and CSS classes. So you can't simply include their CSS and assets in a page, and viola! your site is themed! You have to also modify your site markup to match HTML structure / css classes they expect.

That means you can't simply switch between themes, without modifying Layout.cshtml and similar files in StartSharp. We can't modify Serenity / StartSharp source code to match their specific expectations. It would be vendor / theme lock-in.

Another issue is they don't have any CSS for the components we use like SlickGrid, so we would have to create custom rules for them that matches the theme style. And when a new version comes out, we would also need to maintain our custom CSS.

So in the end, we decided to go with designing our themes for now, which you could simply switch using the settings sidebar.

In the future, we might be offering some guide to integrate such popular themes in case you would still want to use one.

## Our New Themes

Initially we have 6 new themes:

* Azure
* Azure Light
* Cosmos
* Cosmos Light
* Glassy
* Glassy Light

You might have a look at our new themes in our demo at http://serenity.is/demo

> Please note that they are only available for StartSharp customers

They are stable but not considered final, and we might still do some little touches on them where we see fit.

> We're also working on more themes, and improving styling on grid / dialogs / buttons etc.

Here are some screenshots

### Azure Light (default theme for StartSharp)

![Azure Light](img/2018-02-13/azure-light.png)

![Azure Light Panel](img/2018-02-13/azure-light-panel.png)

### Azure (dark sidebar)

![Azure Dark](img/2018-02-13/azure-dark.png)

### Cosmos

![Cosmos](img/2018-02-13/cosmos-dark.png)

### Cosmos Light

![Cosmos Light](img/2018-02-13/cosmos-light.png)

### Glassy (translucent theme with background)

![Glassy](img/2018-02-13/glassy-dark.png)

### Glassy Light

![Glassy Light](img/2018-02-13/glassy-light.png)

## How to Integrate These Themes To Your Project

As you create a new StartSharp project with version 3.4.4+ you'll be getting required CSS files and themes will be available there.

But if you have an application that is created with an older version of StartSharp, please follow these steps:

* Download and install latest StartSharp release from GitHub repository or Serenity.is members area.
* Create a new project to have access to required .less files. Another option is to manually download them from StartSharp repository.
* Using file explorer, copy the skins folder under ~/Content directory (which is under wwwroot for .NET Core projects, project root for others):

![Skins folder](img/2018-02-13/skins-folder.png)

* Open your existing project in Visual Studio and paste *skins* folder under *Content* directory using solution explorer.

* Edit *site.theme.less* file under *~/Content/site/* and insert lines starting with *../skins/* below at the location shown:

```less
//...
@import "../adminlte/social-widgets.less";
@import "../adminlte/skins/_all-skins.less";
@import "../skins/azure.less";
@import "../skins/cosmos.less";
@import "../skins/glassy.less";
@import "../adminlte/mailbox.less";
@import "../adminlte/lockscreen.less";
```

* Edit *ThemeSelection.ts* file under *~/Modules/Common/Navigation*, and copy paste contents from new StartSharp project, to add these options to theme selection dropdown (make sure to replace StartSharp with your project name in that file).

* To change default theme (if you like), edit *_Layout.cshtml* file under *~/Views/Shared/*:

```cshtml
    var theme = themeCookie != null && 
         !themeCookie.IsEmptyOrNull() ? themeCookie : "azure-light";
```

## Where To Next?

We are working on more themes, and customizing grid / table / button etc. styles to match with these themes.

Once Bootstrap 4 is released (stable) we're also planning to update AdminLTE and our new themes.

Please let us know which one of themes you liked most (or didn't like at all, any?) in comments section below.
