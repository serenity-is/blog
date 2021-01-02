---
title: Migrating to Serenity.NET 5
description: Serenity 5 is a major update to Serenity Framework. It only supports .NET 5 and ASP.NET Core 5. For those who missed the news, .NET 5 is actually .NET Core 5, and is simply called .NET now. They skipped version 4 to avoid confusion with legacy .NET Framework 4.
author: Volkan Ceylan
---

# Introduction

From this point on there will only be an .NET 5 / ASP.NET Core 5 version of Serenity / Serene and StartSharp. As announced about six months ago (https://github.com/serenity-is/Serenity/issues/5060) we are no longer supporting .ASP.NET MVC or .NET Framework 4.5. There is a v3 branch in Serenity, Serene and StartSharp repositories but we will not be releasing any new versions there unless we find a critical security issue or similar. Anyway if we release a version there, it will be 3.x not 5.x.

# Background

When we first released .NET Core version of Serenity initial target was to provide easy migration and code compability between ASP.NET MVC and ASP.NET Core templates, so that we could easily share code between two. For example, if we added a new sample to ASP.NET MVC, it had to be easily portable to .NET Core version with as few changes as possible. At that time, .NET Core itself was at its early stages, had some issues, missed some features and was not so stable. So the main branch of development for Serenity was still ASP.NET MVC. This changed in the last year, as we started to develop new features in .NET Core and ported them back to .NET MVC afterwards.  Also, most of our customers we directly worked, ported their projects to .NET Core. 

As the initial target was compability between two, we could not use some features of .NET Core like, primary one being dependency injection at the start. By time it became a pain source, as it is very important for testability and integration with the .NET Core platform itself. 

# List of Changes

We decided to completely embrace .NET Core / ASP.NET Core features, and use them instead of the ones in Serenity where possible, so there are many changes:

- Removed "Dependency" class which was a service locator abstraction, and used Dependency Injection (DI) instead (https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-5.0)
- Removed Config class and used Options pattern where possible (https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-5.0)
- There is almost no static classes / state in Serenity framework now
- Replaced "ILocalCache" interface with .NET Core's "IMemoryCache"
- Replaced Serenity specific "IDistributedCache" interface and their implementations Serenity.Caching.Redis / Serenity.Caching.Couchbase with .NET Core's "IDistributedCache"
- Removed "IAuthenticationService" interface and "AuthenticationService" static class, introduced an injectable "IUserAccessor" abstraction
- Removed Serenity specific "Log" class, and used .NET Core's own logging system
- Replaced ExtensibilityHelper.SelfAssemblies with a ITypeSource abstraction
- Replaced static SqlConnections with ISqlConnections abstraction, it is now theorically possible to use dynamic connection strings per request (multi tenancy++)
- Use DI with lookup scripts, data scripts etc.
- Introduced IRequestContext for service handlers
- Row base class is replaced with IRow interface, and there is a generic Row< TFields > base class with access to its Fields type
- Rows can theorically have different set of custom fields and attributes per request (multi tenancy++)
- Service behaviors rewritten for DI and they can get constructor injection
- Script/CSS bundling use options pattern, and bundles can be specified at runtime, also IFileProvider of .NET used so non-physical files can be included in bundles.
- Default sql dialect can be set per async context
- Redesigned upload system, opens way to use different storage providers like Azure, S3 etc.
- Rewrote core script library with modular typescript

This is not a complete list but should provide an overall idea.

# Serenity 5 Packages

The Serenity 5 packages are prefixed with ".Net", to prevent confusion with v3 packages, and also to avoid users who still use v3 or ASP.NET projects to upgrade by mistake. Here is a list of Serenity 5 packages, and their original names in v3:

- Serenity.Net.Core (formerly Serenity.Core)
- Serenity.Net.Data (formerly Serenity.Data)
- Serenity.Net.Entities (formerly Serenity.Data.Entities)
- Serenity.Net.Services (formerly Serenity.Services)
- Serenity.Net.Web (formerly Serenity.Web or Serenity.Web.AspNetCore)
- Serenity.Scripts (this is not renamed yet as it is a static content package, they are compatible for now but stay at v3 version of Serenity.Scripts if you use 3.x)
- Serenity.Assets (formerly Serenity.Web.Assets)

# Upgrading to Serenity 5

The recommended way to use Serenity 5 / .NET 5 is to download latest StartSharp template, and create a new project.

For existing projects that you would like to port to .NET 5, you'll need to follow steps outlined in:

- https://serenity.is/docs/migration/v3-to-v5

> If you haven't yet migrated your project to .NET Core, first follow steps here:
> https://serenity.is/docs/migration/mvc-to-core

For StartSharp customers, who recently created a project using the prior 3.14.5.6 (or a recent one) we prepared a tool called **stargen** to make migration easier.

See this document for information about **stargen** and using it to migrate your project.

- https://serenity.is/docs/migration/stargen