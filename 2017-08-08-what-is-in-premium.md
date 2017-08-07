---
title: What is in Premium?
description: We provide basic Serenity support in GitHub repository with valuable help from community members. For our users who require more, we offer direct support from Serenity authors. Let's see what you get by purchasing premium support packages.
author: Volkan Ceylan
---

# Premium Support Packages

Our premium support packages are available at [http://serenity.is](http://serenity.is). Currently, three support levels are offered: _Personal_, _Professional_ and _Enterprise_.

All support levels provides access to our *StartSharp* template, which includes additional samples, features and modules (in addition to everything our free template *Serene*), that we'll talk about shortly.

Let's first explain what you get with these support packages.

## Commercial Licence

Even though, Serenity with its permissive MIT licence allows usage in proprietary applications, some companies still require a commercial licence for legal issues. 

*StartSharp* template comes with a commercial licence, that allows you to use it with any number of applications / sites.


## Private Support Desk

You get prioritized e-mail support from our core developers, with guaranteed initial response time of 72 / 48 / 24 hours (excluding weekends and public holidays) based on your support level.

Personal level is only suitable for one developer as it allows 2 incidents per month.

Professional level supports 5 developers and 5 incidents per month.

> These incident limits are not strict and exists as a fair use policy.

## Live Credits

These are hours that you can use to get direct remote support from our developers. 

We recommend using these credits for troubleshooting, so that you can save your precious hours / days by letting us handle the problem in a few minutes.

You may also use these hours for custom development, additional features etc.

It's also possible to buy additional hours, and our premium members get a discount over average fee.

# StartSharp

*StartSharp* is premium version of our free *Serene* template. 

It includes everything Serene has, in addition to extra samples, features and modules.

We're regularly adding extra content into *StartSharp*. 

> Some of these like *card view* are sponsored by our customers. We'd like to thank them for letting us share them with community.

Here is a list of what is currently available in StartSharp...

## Background Task System

You might want to run some process periodically, e.g. daily, or every three hours. 

Even though you could use Scheduled Tasks in Windows, this would require manually configuring such tasks in every server you install your application. 

Windows service is another option but comes with its own set of problems.

StartSharp includes a background task system for your web applications. Even though it is not as feature filled as some other choices like HangFire (https://www.hangfire.io/), it gets the job done.

```cs
namespace StartSharp.Common.Services
{
    using Entities;
    using Serenity;
    using Serenity.Data;
    using System;

    public class MailingBackgroundTask : PeriodicBackgroundTask
    {
        protected override TimeSpan GetInterval()
        {
            var config = Config.Get<MailingServiceSettings>();
            return TimeSpan.FromSeconds(config.Interval);
        }

        protected override void InternalRun()
        {
            var env = Config.Get<EnvironmentSettings>();
            var config = Config.Get<MailingServiceSettings>();

            if (!config.Enabled)
                return;

            using (var connection = SqlConnections.NewFor<MailRow>())
            {
                var m = MailRow.Fields;

                var mailList = connection.List<MailRow>(q => q
                    .Take(config.BatchSize)
                    .Select(m.MailId)
                    //...
```

We implemented our next feature, Background Mail Queue on this task system.


## Background Mail Queue

Sometimes you might want to sent lots of e-mails, like bulletins. Sending thousands of e-mails might take considerable amount time, and the user who tries to publish these e-mails might get timeouts or have to wait a long long time.

Even though you need to send one or two e-mails, if your SMTP server is having problems, your users might get application errors.

Our mail queue lets you to send e-mails in background, retry sending in case of an error and see status of every individual message in a reporting page.

![Mail Queue](img/2017-08-08/mail-queue.png)



## Card View

Card view allows you to show records as fully customizable cards, and easily switch between Card / List views.

> thanks a lot to Brainweber Inc. for sponsoring this feature and letting us share it with community

![Mail Queue](img/2017-08-08/card-view.png)

This feature is implemented as a Mixin you that you can simply apply to any of your grids (EntityGrid / DataGrid).

We used *Vue.js* ([https://vuejs.org/](https://vuejs.org/)) for card templates, so you can simply customize it to your needs and use any property available in your rows.

![Vue JS](img/2017-08-08/vue-js.png)

```ts
new Serenity.CardViewMixin({
                grid: this,
                itemTemplate: `
<table>
    <tr><td rowspan="4" class="img"><img :src="getRandomImage(item)" /></td></tr>
    <tr><td class="name"><a href="#" @click.prevent="editItem(item)">
        {{item.CustomerID}} - {{item.CompanyName}}</a></td>
    </tr>
    <tr><td class="contact">{{item.ContactName}}, {{item.ContactTitle}}</td></tr>
    <tr>
        <td class="country">{{item.City}}, {{item.Country}}</td>
    </tr>
</table>`,
                methods: {
                    editItem: function (item: CustomerRow) {
                        self.editItem(item.ID);
                    },
                    // ...
                }
            });
        }
```

> We really like the way Vue.js works and looking forward to integrate it deeply into Serenity.

## Favorite Views

Serenity support persisting grid settings like visible columns, widths, sort orders. It remembers the latest settings and restores them next time you enter that page. 

Sometimes you might want to save a view and restore it at a later time. This is also useful for dynamic reporting / data export purposes.

![Favorite Views](img/2017-08-08/favorite-views.png)

This is also implemeted as a grid mixin so it can be easily applied to any grid. As a bonus, it is also compatible with card view feature, so it remembers if grid is in card/list view mode and restores that state.


## Data Explorer

This is a table explorer, similar to view data in SQL Management Studio.

It's main purpose is to demonstrate how you could work with dynamic data sources and columns in Serenity.

![Data Explorer](img/2017-08-08/data-explorer.png)


## Data Tables Integration

Serenity use *SlickGrid* under the hood for your listing pages. It is possible to use another grid / table component for some pages if required.

In this sample we demonstrate how you could integrate popular DataTables plugin, in both client side / server side data source mode, while still making use of Serenity services.

![Data Tables](img/2017-08-08/data-tables.png)

## Editors with Inline Actions

One of our premium customers asked us how to add inline action buttons next to editors. Instead of helping him in private, we choose to add this sample to *StartSharp*, so that other members can make use of it. 

> Many samples in *StartSharp* are implemented based on customer requests and feedback.

![Data Tables](img/2017-08-08/editors-inline-actions.png)

## Product Picker Dialog

Even though Select2 allows you to select single or multiple items easily, sometimes you might want to open a dialog and let user to pick items there by marking checkboxes.

![Product Picker](img/2017-08-08/product-picker.png)

> Thanks a lot to Bob Munn for sponsoring this feature and letting us to share it with community. And please get well soon...

## Bootstrap Table / Form

Sergen generates a standardized listing / editing interface and services for your tables, with minimal code possible.

In some cases, you might want to use good old methods like a pure Bootstrap table / form with basic HTTP POST and no AJAX / web services.

![Bootstrap Table](img/2017-08-08/bootstrap-table.png)

This sample is implemented in a way very similar to Entity Framework tutorial you can find on MSDN, whilst using Bootstrap instead of raw HTML tables / forms and Serenity data layer instead of EF.

![Bootstrap Form](img/2017-08-08/bootstrap-form.png)

## Long Running Action with Progress

Even though *Cancellable Bulk Action* sample in Serene is fine for work that you can split into chunks, sometimes you may want to handle a long running operation in one transaction / request.

This sample demonstrates how we could execute a task at server side, possibly in a background thread, return immediately from service action, while still reporting status / progress and allowing user to cancel it if possible.

![Long Running Action](img/2017-08-08/long-running-action.png)

## Conclusion

Premium packages are mainly about support. Buy purchasing one of these support packages, we'll help you save your most precious resource, *time*, you'll get access to extra samples and as a bonus help Serenity moving forward.

So we hope that this will be a win-win. Thanks.