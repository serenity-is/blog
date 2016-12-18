---
title: Don't Repeat Your Permissions (2.7.0)
description: DRY is one of software design principles that we try to closely follow. Among the limited places where we violate this principle was Service Endpoints and Listing Pages, where we repeat permissions and connection keys that are already available at row level. Starting with 2.7.0 that is going to change.
author: Volkan Ceylan
---

## Row Level Permissions

We take security seriously with Serenity applications as it should be with any web application. Permission system is an integrated part of the platform, and it starts with the permissions defined at row level:

```cs
[ReadPermission("Administration:General")]
[ModifyPermission("Administration:Security")]
public sealed class UserRow : LoggingRow, IIdRow, INameRow, IIsActiveRow
{
}
```

Here we specify that anyone who is going to retrieve / list this row has to be granted *Administration:General* permission. To insert / update / delete a row user needs *Administration:Security* permission.

There are other attributes you could use instead of *ModifyPermission*, like *InsertPermission*, *UpdatePermission* and  *DeletePermission* which allows more granular control over operation type and corresponding permission.

As a sample, we used harcoded strings, but it is also possible, and recommended to create a PermissionKeys class, and reference these keys from there, e.g:

```cs
public class PermissionKeys
{
    public const string General = "Administration:General";
    public const string Security = "Administration:Security";
}
```

```cs    
[ReadPermission(PermissionKeys.General)]
[ModifyPermission(PermissionKeys.Security)]
public sealed class UserRow : LoggingRow, IIdRow, INameRow, IIsActiveRow
{
}
```

## Permissions are Checked at Handler Level

Retrieve, List, Create, Update, Delete service requests on repository level are processed by  Serenity Handlers, e.g. ListRequestHandler, SaveRequestHandler, DeleteRequestHandler etc.

You should have below lines in your Repository.cs:

```cs
private class MySaveHandler : SaveRequestHandler<MyRow> { }
private class MyDeleteHandler : DeleteRequestHandler<MyRow> { }
private class MyRetrieveHandler : RetrieveRequestHandler<MyRow> { }
private class MyListHandler : ListRequestHandler<MyRow> { }
```

These are subclasses of those handlers specific to your Row type.

It is critical to know that the point where we check row permissions IS inside these handlers.

If you use a DBCommand, DataReader, Dapper, Connection.UpdateById etc. extensions directly, no checks occurs.

The reasoning behind this is that they operate at a lower level and we can't and shouldn't intercept these methods.

Sometimes you might also be doing some updates / deletes on behalf of the consumer (user), and there is no way to know if that is the case.

## Permission Checks in Listing Pages

Here is the source for User listing page, which is common in Serenity applications:

```cs
[RoutePrefix("Administration/User"), Route("{action=index}")]
public class UserController : Controller
{
    [PageAuthorize(PermissionKeys.General)]
    public ActionResult Index()
    {
        return View(MVC.Views.Administration.User.UserIndex);
    }
}
```

*PageAuthorize* attribute is a specialized type of ASP.NET MVC *Authorize* attribute, accepting permission keys to check.

While adding this page to navigation we reference it with its class name:

```cs
[assembly: NavigationLink(9400, "Administration/User Management",
    typeof(Administration.UserController), icon: "icon-people")]
```

Navigation system checks the controller class to determine which permission key to check for showing this page in navigation. So, users without *PermissionKeys.General* permission won't see that page. Navigation system *doesn't duplicate* this *information*, so we need to specify it only once.

But we should notice that, our UserRow, corresponding to UserController already has a *ReadPermission* attribute. If one day, we change the read permission for UserRow, we'll have to change the one on *UserController Index* action too. Otherwise, we'll get a service error after entering the page.

There might be some edge cases where a listing pages permission doesn't have to match the one on its corresponding row, but 99% percent of time they do.

Starting with 2.7.0, you'll be able to *reuse* ReadPermission information on a row, to set permission for its listing page, e.g.:

```cs
[RoutePrefix("Administration/User"), Route("{action=index}")]
public class UserController : Controller
{
    [PageAuthorize(typeof(Entities.UserRow))]
    public ActionResult Index()
    {
        return View(MVC.Views.Administration.User.UserIndex);
    }
}
```

This might seem like a not so important feature, but we got questions like *I've changed permission on my row, but my page is not shown is navigation, what should i do?* so many times you can't imagine.

> You can still use old way of hard coding permission keys if you like. But this will be the code Sergen generates by default.

## Permission Checks in Service Endpoints

There is another point we check permissions before a service request reaches the service handlers.

It is your service endpoint like this one:

```cs
[RoutePrefix("Services/Administration/User"), Route("{action}")]
[ConnectionKey("Default"), ServiceAuthorize(PermissionKeys.General)]
public class UserController : ServiceEndpoint
{
    [HttpPost]
    public SaveResponse Create(IUnitOfWork uow, SaveRequest<MyRow> request) {

    }
    //...
}
```

ServiceAuthorize is also similar to PageAuthorize, but instead of redirecting the user to login page when user doesn't have the permission, or showing the yellow error page, it returns a proper ServiceError in JSON format.

All UserController actions requires *Administration:General* permission because of  ServiceAuthorize attribute on top of the type.

Again, we are repeating information already available in our UserRow.

As endpoint code is generated by Sergen, most users are not aware of the check here, so we receive similar questions to one about PageAuthorize. *Auto* might sometimes mean *obscure*.

You might think that *why do we have to put an extra check here, if request handlers already does that?*.

Problem is that developer might add a new method here that doesn't use our generic handlers.

```cs
public class UserController : ServiceEndpoint
{
    [HttpPost]
    public ServiceResponse FormatMyDisk(IUnitOfWork uow, FormatMyDiskRequest request)
    {    
    }
    //...
}
```

If developer doesn't know or forgot that permission checks are done by service handlers, he might assume this method can only be called by an administrator. I hear you say, *come on nobody does that*.

All devs in my team did that mistake once. That's why i made sure Sergen adds ServiceAuthorize attribute on service endpoints.

There is also another *repeat* we did here. ConnectionKey attribute could also reuse information on UserRow. Let's fix both with 2.7.0:

```cs
[RoutePrefix("Services/Administration/User"), Route("{action}")]
[ConnectionKey(typeof(MyRow)), ServiceAuthorize(typeof(MyRow))]
public class UserController : ServiceEndpoint
{
}
```

Ok now both the connection key and permission key is read from *MyRow*, which is *UserRow* for this controller.

One more thing to handle is, if we leave it like this, Create, Update, Delete methods will also check *ReadPermission* attribute on row. But, these handlers doesn't use *ReadPermission* if there is a *ModifyPermission*, *InsertPermission*, *DeletePermission* etc. on row.

We now also have, some subclasses of *ServiceAuthorize* attribute that handles these special cases similar to corresponding service handlers:

* AuthorizeUpdateAttribute
* AuthorizeDeleteAttribute
* AuthorizeCreateAttribute

By making use of these new attributes, our endpoint now becomes:

```cs
[RoutePrefix("Services/Administration/User"), Route("{action}")]
[ConnectionKey(typeof(MyRow)), ServiceAuthorize(typeof(MyRow))]
public class UserController : ServiceEndpoint
{
    [HttpPost, AuthorizeCreate(typeof(MyRow))]
    public SaveResponse Create(IUnitOfWork uow, SaveRequest<MyRow> request)
    {
        return new MyRepository().Create(uow, request);
    }

    [HttpPost, AuthorizeUpdate(typeof(MyRow))]
    public SaveResponse Update(IUnitOfWork uow, SaveRequest<MyRow> request)
    {
        return new MyRepository().Update(uow, request);
    }

    [HttpPost, AuthorizeDelete(typeof(MyRow))]
    public DeleteResponse Delete(IUnitOfWork uow, DeleteRequest request)
    {
        return new MyRepository().Delete(uow, request);
    }

    public RetrieveResponse<MyRow> Retrieve(IDbConnection connection, RetrieveRequest request)
    {
        return new MyRepository().Retrieve(connection, request);
    }

    public ListResponse<MyRow> List(IDbConnection connection, ListRequest request)
    {
        return new MyRepository().List(connection, request);
    }
}
```

This is more secure than before, and we don't repeat any permission information already available on UserRow.

All endpoints in Serene are updated to use this new feature. Sergen also produces compatible code.

Let's DRY.
