---
title: Sqlite Support in Serene
description: Serene had support for SqlServer, MySql, Postgres and Oracle for some time. Now it is time to welcome Sqlite. 
author: Volkan Ceylan
---

SqliteDialect has long been in Serenity and i know some users who used Sqlite internally, 
but we had no official support in Serene.

As Asp.Net Core version of Serene is about to launch soon, SQL LocalDB we used so far 
by default was no feasible option in MAC/Linux.

What i needed was some database that could work in Window/MAC/Linux and require 
no installation if possible.

Sqlite looked like a natural choice, so i decided to give it a go. 

> I tried this on Windows/.NET45 so it won't work as is in .NET Core yet.

## Registering Sqlite Provider

MySQL has a .NET provider named System.Data.Sqlite. You need to first install it in 
MyProject.Web:

```
Install-Package System.Data.SQLite.Core
```

If you didn't install this provider in GAC/machine.config before, or don't want to install it there, you need to register it in web.config file:

```xml
<configuration>
  // ...
  <system.data>
    <DbProviderFactories>
        <remove invariant="System.Data.SQLite"/>
        <add name="SQLite Data Provider"
          invariant="System.Data.SQLite"
          description=".Net Framework Data Provider for SQLite"
          type="System.Data.SQLite.SQLiteFactory, System.Data.SQLite"/>
    </DbProviderFactories>
  </system.data>
  // ...
```

## Setting Connection Strings

Next step is to replace connection strings for databases you want to use with MySql:

```xml
  <connectionStrings>
    <add name="Default" connectionString=
         "Data Source=|DataDirectory|Serene_Default_v1.sqlite;" 
      providerName="System.Data.Sqlite" />
    <add name="Northwind" connectionString=
        "Data Source=|DataDirectory|Serene_Northwind_v1.sqlite;" 
      providerName="System.Data.Sqlite" />
  </connectionStrings>
```

> Replace *Serene* with your solution name, e.g. *MyApplication*.

If you are using Serene 2.8.1+, this is all you will have to do (not released as of writing).

You won't have to do following changes if you use a newer template.

## Changes in *SiteInitialization.Migrations.cs*

As we didn't have any official support in Serene, i had to do some changes in 
*SiteInitialization.Migrations.cs* file.

This one is to create a new Sqlite database file under *~/App_Data/* directory if none exists:

```cs
private static void EnsureDatabase(string databaseKey)
{
    var cs = SqlConnections.GetConnectionString(databaseKey);

    var serverType = cs.Dialect.ServerType;
    bool isSql = serverType.StartsWith("SqlServer",
         StringComparison.OrdinalIgnoreCase);
    bool isPostgres = !isSql & serverType.StartsWith("Postgres", 
        StringComparison.OrdinalIgnoreCase);
    bool isMySql = !isSql && !isPostgres && serverType.StartsWith("MySql", 
        StringComparison.OrdinalIgnoreCase);
    bool isSqlite = !isSql && !isPostgres && !isMySql && 
        serverType.StartsWith("Sqlite", StringComparison.OrdinalIgnoreCase);
    if (!isSql && !isPostgres && !isMySql && !isSqlite)
        return;

    var cb = cs.ProviderFactory.CreateConnectionStringBuilder();
    cb.ConnectionString = cs.ConnectionString;
    string catalogKey = "?";

    if (isSqlite)
    {
        catalogKey = "Data Source";
        if (!cb.ContainsKey(catalogKey))
            return;

        var dataFile = cb[catalogKey] as string;
        if (string.IsNullOrEmpty(dataFile))
            return;

        dataFile = dataFile.Replace("|DataDirectory|", 
            HostingEnvironment.MapPath("~/App_Data/"));
        if (File.Exists(dataFile))
            return;

        Directory.CreateDirectory(Path.GetDirectoryName(dataFile));
        using (var sqliteConn = SqlConnections.New(cb.ConnectionString, cs.ProviderName))
        {
            var createFile = ((WrappedConnection)sqliteConn).ActualConnection.GetType()
                .GetMethod("CreateFile", BindingFlags.Static);
            if (createFile != null)
                createFile.Invoke(null, new object[] { dataFile });
        }
            
        SqlConnection.ClearAllPools();
        return;
    }

    //...
}
```
EnsureDatabase has code to create databases for SqlServer, Oracle, Postgres, MySql etc. 
Here we add a specialized one for Sqlite.

> I didn't want to add a direct dependency to *System.Data.SqliteConnection* 
assembly in Serene yet, so that's why i had to use reflection to call *CreateFile* 
static method on it.

## Adding Migrations for Northwind

Serene *Default* database completely uses migrations, so it should work 
out of the box without any changes.

But *Northwind* is another story. Serene has specialized scripts for every database
type, so we had to add one for Sqlite too.

I found a script for *Northwind/Sqlite* by *Valon Hoti/Len Boyette*, and had to modify it a bit:

https://github.com/volkanceylan/Serene/blob/master/Serene/Serene.Web/Migrations/NorthwindDB/NorthwindDBScript_Sqlite.sql

You need to set its build action to *Embedded Resource*, and do following change
in *NorthwindDB_20141123_155100_Initial.cs*:

```cs
[Migration(20141123155100)]
public class DefaultDB_20141123_155100_Initial : Migration
{
    public override void Up()
    {

        //...
        IfDatabase("Oracle")
            .Execute.EmbeddedScript("Serene.Migrations.NorthwindDB.NorthwindDBScript_Oracle.sql");

        IfDatabase("Sqlite")
            .Execute.EmbeddedScript("Serene.Migrations.NorthwindDB.NorthwindDBScript_Sqlite.sql");

        //...
    }
```

That should be enough to bring Sqlite support to Serene.

Running should be enough to get your Sqlite databases created and populated.

But there is one more problem we'll need to handle. I'll leave it for the next article.