---
title: Sergen for .NET Core - Part 1
description: Sergen for .NET Core works as a console application, instead of a WPF based GUI as it has to support platforms like OSX / Linux in addition to Windows. It has only support for Sql Server yet, but we are working on other database types. 
author: Volkan Ceylan
---

## Sergen for ASP.NET MVC (written in WPF)

Sergen, which is our code generator for Serenity, originally only worked and designed to work with Sql Server. 
By time, some users requested other database types and we gradually added support for them. But, code for Sergen became a mess that i don't like at all.

Here is some code snippet (which i'm certainly not proud of) from the original Sergen, where it tries to determine 
foreign keys of a table:

```cs

public static List<ForeignKeyInfo> GetTableSingleFieldForeignKeys(IDbConnection connection, string schema, string tableName)
{
    var inf = InformationSchema(connection);
    List<ForeignKeyInfo> foreignKeyInfos = new List<ForeignKeyInfo>();

    if (connection.GetDialect().ServerType.StartsWith("Sqlite", StringComparison.OrdinalIgnoreCase))
    {
        try
        {
            using (var reader = SqlHelper.ExecuteReader(connection, (String.Format("PRAGMA foreign_key_list({0});", tableName))))
            {
                while (reader.Read())
                {
                   //...
                }
            }
        }
        catch (Exception)
        {
        }

        return foreignKeyInfos;
    }

    if (connection.GetDialect().ServerType.StartsWith("Firebird", StringComparison.OrdinalIgnoreCase))
    {
        var query = @"
select 
 PK.RDB$RELATION_NAME as PKTABLE_NAME
,ISP.RDB$FIELD_NAME as PKCOLUMN_NAME
//...
order by 1, 5";

        try
        {
            using (var reader = SqlHelper.ExecuteReader(connection, (String.Format(query, tableName))))
            {
                // ...
            }
        }
        catch (Exception)
        {
        }

        return foreignKeyInfos;
    }

    if (connection.GetDialect().ServerType.StartsWith("Postgres", StringComparison.OrdinalIgnoreCase))
    {
        // ...
    }

    if (connection.GetDialect().ServerType.StartsWith("MySql", StringComparison.OrdinalIgnoreCase))
    {
        // ...
    }

    if (connection.GetDialect().ServerType.StartsWith("SqlServer", StringComparison.OrdinalIgnoreCase))
    {
        // ...
    }

    return foreignKeyInfos;
}
```

Yes it's a real mess, and a sample for how you *shouldn't* write code, but that mess is still able to work 
properly with MS Sql Server, Postgres, MySql, Sqlite and Firebird, while providing a nice WPF based UI.

## Designing Sergen for .NET Core

Sergen for .NET Core, which is a complete rewrite as a console application, only works with Sql Server yet.

To support other database types, while not allowing the transformation to a spaghetti code like before, 
we decided to go with a provider based model.

Here is the provider interface Sergen requires to work with an arbitrary database type:

```cs
public interface ISchemaProvider
{
    IEnumerable<TableName> GetTableNames(IDbConnection connection);
    string DefaultSchema { get; }
    IEnumerable<string> GetPrimaryKeyFields(IDbConnection connection, string schema, string table);
    IEnumerable<string> GetIdentityFields(IDbConnection connection, string schema, string table);
    IEnumerable<ForeignKeyInfo> GetForeignKeys(IDbConnection connection, string schema, string table);
    IEnumerable<FieldInfo> GetFieldInfos(IDbConnection connection, string schema, string table);
}
```

We have to create one class per database type that implements this interface. For example, MySqlSchemaProvider,
SqliteSchemaProvider...

It's not an easy task, as i had to install all these servers in my PC, create sample databases like Northwind on them,
and test suggested methods i could find from their documentation, Google and Stack Overflow.

### GetTableNames Method

Unfortunately, there is no universal way to query a database for its metadata, not even a list of its tables.

There is some standard called *INFORMATION_SCHEMA* which is supposed to be supported by all database types, but that's not
the actual case.

*GetTableNames* implementations should return a list of this simple class:

```cs
public class TableName
{
    public string Schema { get; set; }
    public string Table { get; set; }
    public bool IsView { get; set; }

    public string Tablename
    {
        get { return Schema.IsEmptyOrNull() ? Table : Schema + "." + Table; }
    }
}
```

Here is how we do it with some database types:


**MS Sql Server:**

```cs
public IEnumerable<TableName> GetTableNames(IDbConnection connection)
{
    return connection.Query(
            "SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE FROM INFORMATION_SCHEMA.TABLES " +
            "ORDER BY TABLE_SCHEMA, TABLE_NAME")
        .Select(x => new TableName
        {
            Schema = x.TABLE_SCHEMA,
            Table = x.TABLE_NAME,
            IsView = x.TABLE_TYPE == "VIEW"
        });
}
```

MS Sql supports schemas (e.g. **dbo**) and for views this query returns *VIEW* in its *TABLE_TYPE* column.


**Firebird:**

```cs
public IEnumerable<TableName> GetTableNames(IDbConnection connection)
{
    return connection.Query(@"
            SELECT RDB$RELATION_NAME NAME, RDB$VIEW_BLR ISVIEW
            FROM RDB$RELATIONS  
            WHERE (RDB$SYSTEM_FLAG IS NULL OR RDB$SYSTEM_FLAG = 0)")
        .Select(x => new TableName
        {
            Table = StringHelper.TrimToNull(x.NAME),
            IsView = x.ISVIEW != null
        });
}
```

Firebird doesn't support database schemas, so we can only read table name and view flag.

**MySql:**

```cs
public IEnumerable<TableName> GetTableNames(IDbConnection connection)
{
    return connection.Query(
            "SELECT TABLE_NAME, TABLE_TYPE FROM INFORMATION_SCHEMA.TABLES " +
            "WHERE TABLE_SCHEMA = Database() " +
            "ORDER BY TABLE_SCHEMA, TABLE_NAME")
        .Select(x => new TableName
        {
            Table = x.TABLE_NAME,
            IsView = x.TABLE_TYPE == "VIEW"
        });
}
```

MySql doesn't actually support schemas but it treats databases as if they are schemas. 
So we don't have any schema information here again. We have to filter results by current database, or otherwise
we'll get table list of other databases as well.

> Sergen .NET Core version won't work with MySql yet, as its .NET Core provider is in beta, and NuGet doesn't let me
add a dependency to a preview version of a NuGet package, as Sergen itself is not preview. Let's hope they'll get out
of beta soon.

**Oracle:**

```cs
public IEnumerable<TableName> GetTableNames(IDbConnection connection)
{
    return connection.Query("SELECT owner, table_name FROM all_tables")
        .Select(x => new TableName
        {
            Schema = x.owner,
            Table = x.table_name
        });
}
```

Oracle doesn't even have a notion of separate *databases*. I made mistake of trying to create a separate DATABASE in 
Oracle which barely corresponds to a *Sql Server Instance*.

> Oracle was the hardest one while i was first trying to port Serene to it. And i never tried to port Sergen, as i was so tired.

It has *users* which you may think like databases.
We may call it a *user oriented* database system. Here *owner* is actually the user who owns the table.

> Sergen .NET Core version won't work with Oracle yet, as there is no free provider for .NET CORE.

**Postgres:**

```cs
public IEnumerable<TableName> GetTableNames(IDbConnection connection)
{
    return connection.Query(
            "SELECT table_schema, table_name, table_type " + 
            "FROM information_schema.tables " +
            "WHERE table_schema NOT IN ('pg_catalog', 'information_schema') " +
            "ORDER BY table_schema, table_name")
        .Select(x => new TableName
        {
            Schema = x.table_schema,
            Table = x.table_name,
            IsView = x.table_type == "VIEW"
        });
}
```

Like, MS Sql, Postgres supports schemas. But it returns tables in its system schemas, unless you filter them
explicitly.

> The challange i had with Postgres was case sensitivity and quoting. But it wasn't actually Postgres fault.
It was due to FluentMigrator Postgres provider quoting all field / table names while it shouldn't. This was clearly
suggested to be avoided in Postgres manual.

**Sqlite:**

```cs
public IEnumerable<TableName> GetTableNames(IDbConnection connection)
{
    return connection.Query(
            "SELECT name, type FROM sqlite_master WHERE type='table' or type='view' " +
            "ORDER BY name")
        .Select(x => new TableName
        {
            Table = x.name,
            IsView = x.type == "view"
        });
}
```

Being a basic database engine, Sqlite doesn't support schemas like most others.

### That's All For Now

In Part 2, we'll talk about how we can determine PRIMARY KEYS / IDENTITY fields.