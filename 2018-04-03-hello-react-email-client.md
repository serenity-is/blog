---
title: Hello to React and Email Client (IMAP) Sample in StartSharp
description: As you possibly already know, Serenity is a declarative framework that is primarily designed for data-heavy business applications, and administrative interfaces of web sites where you have lots of forms and lists. It has a unique widget system which works pretty well for backend but it didn't feel as intuitive for frontend apps. After considering many alternatives like Vue, Angular, and countless other frontend libraries around, our answer is going to be React.
author: Volkan Ceylan
---

## Serenity and Widget System

Serenity widget system was originally based on the idea of jQuery UI widgets but we modified it to basically use classes declared in TypeScript.

We don't want you (or ourselves) to write much UI code, so you can declare those widgets or editors in your form definitions at server side C# code, and we automatically create them for you while generating the UI. This makes it much easier to maintain.

Here is a section of OrderForm.cs we use to declare Order editing interface:

```cs
    [BasedOnRow(typeof(Entities.OrderRow), CheckNames = true)]
    public class OrderForm
    {
        [Tab("General")]
        [Category("Order")]
        public String CustomerID { get; set; }
        [DefaultValue("now")]
        public DateTime OrderDate { get; set; }
        public DateTime RequiredDate { get; set; }
        public Int32? EmployeeID { get; set; }
```

Here we specify that we want these fields under a *General* tab and *Order* category and Serenity automatically creates a property grid with Bootstrap tabs widget and a *General* tab, and also creates a *Category* with caption *General*.

It also creates a *CustomerEditor* widget for CustomerID field (as it's declared on row), *Date* editor widget for *OrderDate* and *RequiredDate*, and *LookupEditor* widget for the *EmployeeID* field (again declared on row itself).

And, in the end you get a magically created complex UI like *Order Dialog*, without actually writing any Javascript / HTML code.

This is close to perfect, and *what we recommend* for all your editing screens. It takes much less time, is easier to maintain, easier to add new features, and all the look across your backend site is consistent.

## The Issue with Frontend / Non Standard Pages

For frontend, and some parts of your applications that are not that standard, Serenity forms and grids may not be that useful though. There, you may have no forms at all, or may need a product listing page that should use ordinary tables instead of SlickGrid.

In that case, you could simply proceed with pure old HTML / Javascript / CSS, possibly using TypeScript and CSHTML. What if you wanted to mix Serenity widgets like lookup editor in some parts of those pages? Even though it is possible to create Serenity widgets in pure javascript, it doesn't feel so natural.

Even though we like Razor (CSHTML) and all the power it brings to ASP.NET applications, it's cumbersome to handle validation, post backs, state, navigation etc. That's one of the reasons why frontend frameworks became so popular.

## Another Problem with Dialog Templates

Frontend aside, some of our dialogs use simple HTML templates to customize their interface to something different than the standard UI. Here is a sample from CustomerDialog:

```html
<div id="~_Tabs" class="s-DialogContent">
    <ul>
        <li><a href="#~_TabInfo"><span>{{text:"Db.Northwind.Customer.EntitySingular"}}</span></a></li>
        <li><a href="#~_TabNotes"><span>{{text:"Db.Northwind.Note.EntityPlural"}}</span></a></li>
        <li><a href="#~_TabOrders"><span>{{text:"Db.Northwind.Order.EntityPlural"}}</span></a></li>
    </ul>
    <div id="~_TabInfo" class="tab-pane s-TabInfo">
        <div id="~_Toolbar" class="s-DialogToolbar">
        </div>
        <div class="s-Form">
            <form id="~_Form" action="">
                <div class="fieldset ui-widget ui-widget-content ui-corner-all">
                    <div id="~_PropertyGrid"></div>
                    <div class="clear"></div>
                </div>
            </form>
        </div>
    </div>
    <div id="~_TabNotes" class="tab-pane s-TabNotes">
    </div>
    <div id="~_TabOrders" class="tab-pane s-TabOrders">
        <div id="~_OrdersGrid">
        </div>
    </div>
</div>
```

Even though templates are powerful, and allows you to customize the UI as you like, the problem here, and with templates in general is maintainability. 

Here we hardcode some IDs and reference them in our dialog like this:

```ts
this.ordersGrid = new CustomerOrdersGrid(this.byId('OrdersGrid'));
```

What if we mistyped the id of the element, *OrdersGrid* in the template or TypeScript code? There is no check at compile time, so we don't get any errors until we open the dialog. Even if browser console might indicate that something is wrong, it might make us lose time to identity the problem.

The same goes with tab keys and their corresponding panes.

Next issue is, we are declaring the placeholder for *OrdersGrid* in the template, but have to create it explicitly in TypeScript code as well. One is better than two.

## Looking at the Frontend Scene

If you make some research in GitHub (my primary source :), and Internet in general most popular frameworks seem to be VueJS, Angular and React followed by some less known ones like Ember, Polymer, Inferno, Preact, Mithril etc. (some are React alternatives.) There are so many and it takes a lot deal of time to look at all the options and decide.

Anyway, there seems to be a consensus on one specific pattern, which is named VDOM (virtual DOM) nowadays, and almost all popular frameworks are based on that. I won't go into details on what VDOM is for now, Google is your friend ;)

## VueJS, Angular or React

Believe me, i looked at almost a hundred fronted framework in last few months, even wrote one myself to understand how they (and VDOM in general) actually work.

It feels better to go with one of popular ones, even if they are not the best, fastest, or smallest, they tend to have better support, more examples and resources.

## VueJS

We used VueJS in StartSharp, for features like card view, Sergen web UI and also for some other projects internally. It is a real gem and getting more and more popular everyday (it's currently number one on GitHub).

Unfortunately, it's tooling support is lacking, especially in Visual Studio.

Even though it could be used without a module loader, e.g. as global namespace, its typings doesn't support that.

It uses a lovely template syntax but you get no compile time checking and have to wait until errors pop runtime.

There is a useful extension for Visual Studio Code editing experience (Vetur), but not for Visual Studio.

It's data / class system doesn't work well with TypeScript, and you get no proper intellisense.

So, at least for the moment, i had to leave it out, as it didn't meet expectations for large scale applications, even if i didn't want to.

## Angular

Angular which is from Google, is written in TypeScript and it could be a very compelling reason to use it with Serenity.

It's considered to be a monolithic framework, which *expects* you to use it for all parts of your application. It doesn't force you to do that, but development experience might not be as good otherwise.

That also means they already took decisions on behalf of yourself, which many considers a good thing. As you won't have to waste time on choosing a router for example.

I can't be counted as an Angular expert, but in my humble opinion their decisions change too much and too often (like what happened from Angular 1 to Angular 2).

What i care about most is backward compability, and interoperability with existing widgets and it didn't seem like it will be trivial to handle that with Angular, so at least for now, we also leave it out.

## React

React, which is from Facebook, is aims to be only the view layer for your application, which i like about it most.

Even though it is not written with TypeScript, it has great support for it. Microsoft seems to be caring about React and there are even some settings in tsconfig.json that are React specific. Visual Studio and Visual Studio Code works equally well.

It supports a format called JSX, which is TSX in TypeScript, that allows you to mix Javascript / HTML in one file. Hey stop there! Isn't it considered to be a bad thing? At first i also thought so, but after using it, changed my mind.

Here is a sample TSX:

```jsx
render() {
    return (
        <form>
            <div className="field">
                <label className="caption" htmlFor="Username">Username</label>
                <StringEditor id="Username" maxLength={20} />
            </div>

            <div className="field">
                <label className="caption" htmlFor="Password">Password</label>
                <PasswordEditor id="Password" />
            </div>

            <div className="field">
                <label className="caption" htmlFor="Tenant">Tenant</label>
                <LookupEditor id="Tenant" lookupKey={TenantRow.lookupKey} />
            </div>

            <div>
                <button className="btn btn-primary" type="submit">Login</button>
            </div>
        </form>
    )
}
```

Ok, it looks like HTML and offers full intellisense and compile time checking but it's not perfect. You have to use *className* instead of *class* and *htmlFor* instead of *for*. These are some decisions taken by React team as they thought *class* and *for* are reserved words and would cause problems in some browsers like IE, but it's no longer relevant. React alternatives like Preact support *class* as well.

Anyway, have you noticed Serenity widgets there? *StringEditor*, *PasswordEditor* and *LookupEditor* to be precise? Yes, they are Serenity widgets and we found a way to make them work with *React/JSX* even if they are not actually *React Components*. And that is a big win.

OK, so can we now use React instead of our dialog templates? Unfortunately no. We could but in that case you shouldn't do any jQuery manipulations on your HTML markup, as React expects to own the DOM. Anyway, it's possible to mix jQuery / React code to some extent, as is the case with our widgets.

## React Alternatives

React is so popular and simple that it has many alternatives. Some of them offer almost %99 compability with React itself.

Here is a list of ones we know and tried:

- Preact: 3KB (gzip), aims to be the smallest one, a bit better than React in speed. Requires a compat package to be almost compatible.
- Inferno: 8KB (gzip), generally fastest one, requires a compat package to be almost compatible
- NervJS: 10KB (gzip), close to Inferno in terms of speed, doesn't require compat package

React itself is around 30KB gzipped, so if it feels too big, it's nice to have alternatives. Theorically, you can just replace a few scripts to switch to an alternative, but they might sometimes be a bit behind in features. For example, latest *Fragment* element introduced in React 16.2 is not supported by any of them as of writing.

## React Support in StartSharp

As of StartSharp 2.6.0, we started integrating React into Serenity, and converted existing Vue based code like CardView into React.

> React support is currently StartSharp only, there are no samples yet in Serene

Here is a class that shows how customer card view in CustomerGrid rendered using React:

```jsx
export class CustomerCard extends React.Component<CustomerCardProps> {
        render(): React.ReactNode {
            var item = this.props.item;
            return (
                <table>
                    <tbody>
                        <tr>
                            <td rowSpan={4} className="img">
                                <img src={getRandomImage(item)} />
                            </td>
                        </tr>
                        <tr>
                            <td className="name">
                                <a href="javascript:;" onClick={e => this.props.editItem(item)}>
                                    {item.CustomerID} - {item.CompanyName}
                                </a>
                            </td>
                        </tr>
                        <tr>
                            <td className="contact">{item.ContactName}, {item.ContactTitle}</td>
                        </tr>
                        <tr>
                            <td className="country">{item.City}, {item.Country}</td>
                        </tr>
                    </tbody>
                </table>
            );
        }
    }
```

Before, we were using VueJS for this, but it had no intellisense / compile time validation.

## EmailClient Sample

StartSharp now comes with a full-featured IMAP based e-mail client written with React. It can connect to any e-mail account supporting IMAP protocol, like Gmail, Homail, Yandex, Yahoo in addition to your local mail server.

We use MailKit for connecting to servers.

Here is some screenshots from the new sample:

![Email List](img/2018-04-03/emaillist.png)

![Email Read](img/2018-04-03/emailread.png)

You can try it in demo, but you may want to use a test e-mail account as you'll need to enter your username and password in login screen of the e-mail client.

> We don't store it but anyway don't trust anyone on such matters

## Updatable StartSharp NuGet Packages

As of 2.6.0, we started separating parts of frontend code into NuGet packages (currently Serenity.Pro.Scripts which contains React based UI components, EmailClient and App code in separate scripts), so that you can now simply update them to get latest features and bug fixes into your existing StartSharp application, just like you do with other Serenity packages.

> Unlike other Serenity packages, these NuGet packages are not publicily available, so you can download them from StartSharp repo and copy into *pro-packages* folder under your solution directory.

## Road Ahead

We're just starting React integration and there are many predicted changes, but we expect them to be fully backward compatible.

Currently React will be only used for some parts / samples but in the future, we're hoping to develop a pure React based UI.