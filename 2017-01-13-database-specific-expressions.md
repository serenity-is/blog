---
title: Database Specific Expressions
description: Serene runs on a variety of databases, including MS Sql, Postgres, MySql, Oracle and now Sqlite as we introduced in yesterdays article. We talk about the challange we had to handle as Sqlite didn't support CONCAT, and how we resolved it by a new Serenity feature.
author: Volkan Ceylan
---

# Switching Database Types in Serene

Serene supports several database types out of the box. This is backed by Serenity with its dialect support.
Switching to another database from the default Sql Server (local db) is usually a three step process:

* Add reference to NuGet package for the provider (e.g. System.Data.Sqlite)
* Register provider in web.config (if not already done by the provider automatically)
* Change connection strings and provider names in web.config

Serenity determines which dialect to use by looking at provider name field of the connection, creates database if required,
runs migrations at application start and now you have Serene / Northwind running on your database of choice.

# Database Agnostic Expressions

For this to work, we tried to handle all code in samples like Northwind in a database agnostic way. 

For example, we didn't use *database schemas*, as some database types didn't support it.

We used SQL server specific brackets by default "[]", but made sure Serenity replaces them with database specific ones
while executing the query using a simple (quick) parser.

Another critical thing was to use expressions that are supported by all databases. For example, for string addition,
all database types has a different set of methods. Sql Server has *"+"* operator and *CONCAT*, while some servers has
*||* operator and *CONCAT*. 

So we preferred to use *CONCAT*. And, while some servers supports two or more operands for this method, some servers support only two, thus we had to assume two.

Unfortunately *CONCAT* isn't supported before Sql Server 2012. We hoped not many still 
uses Sql Server 2000/2005/2008 but there were, and they had to find replace 
CONCAT with "+" in [Expression] attributes.

So far, we managed to write database agnostic expressions like this (with a few exceptions):

```cs
[DisplayName("FullName"), QuickSearch]
[Expression("CONCAT(T0.[FirstName], CONCAT(' ', T0.[LastName]))")]
public String FullName
{
    get { return Fields.FullName[this]; }
    set { Fields.FullName[this] = value; }
}
```

But, our newcomer, Sqlite doesn't support *CONCAT* at all. It only has *"||*". 
We could again tell Sqlite users to find and replace *CONCAT* with *"||"*. 
Well, not pretty.

*CONCAT* isn't the only problem. There might be many different cases, where one can't
write an expression in a database agnostic way.

For example here is how to extract month from a date in different database types:

- **Oracle**: `to_char(TheDate,'mm')` or `EXTRACT(month from TheDate)`
- **Postgres**: `date_part('month', TheDate)` or `EXTRACT(month from TheDate)`
- **MySql**: `month(TheDate)` or `EXTRACT(month from TheDate)`
- **Sqlite**: `select strftime('%m', TheDate)`
- **SqlServer**: `select DATEPART(m, TheDate)` or `MONTH(TheDate)`

There might be other methods. Some are supported by more recent versions etc.

## Resolving the Problem with Dialect Specific Expressions

*ExpressionAttribute* can only accept *constant strings*, so using a method to 
change expression based on database type is not an option, or would be messy.

Our solution to this problem is *dialect targeted expressions* in Serenity 2.8.1:

```cs
[DisplayName("FullName"), QuickSearch]
[Expression("CONCAT(T0.[FirstName], CONCAT(' ', T0.[LastName]))")]
[Expression("(T0.FirstName || ' ' || T0.LastName)", Dialect = "Sqlite")]
public String FullName
{
    get { return Fields.FullName[this]; }
    set { Fields.FullName[this] = value; }
}
```

Here we specify multiple expressions for our *FullName* field. First one applies to
any dialect / database type. Second one applies ony to dialects matching *"Sqlite"*.
Serenity determines which one to use based on connection dialect type.

But which connection?

Remember the *ConnectionKey* attribute on top of our rows?:

```cs
[ConnectionKey("Northwind")]
public sealed class EmployeeRow : Row, IIdRow, INameRow
{
    // ...
}
```

Serenity uses this information if available on top of row to determine the dialect for 
a row. If none exists, the one set in *SqlConnections.DefaultDialect* is used.

# How Dialect Names Are Matched

We have following types of dialects:

- FirebirdDialect
- MySqlDialect
- OracleDialect
- PostgresDialect
- SqliteDialect
- SqlServer2000Dialect
- SqlServer2005Dialect
- SqlServer2008Dialect
- SqlServer2012Dialect

You can always write your own custom dialect if required, but these are the ones 
that comes out of the box.

Matching a dialect string in *Expression* attribute is first performed on class name
of the dialect. It uses a *starts with* operator, so if we wrote expression like this:

```
[Expression("...", Dialect = "Sql")]
```

this would match *SqliteDialect*, *SqlServer2000Dialect*, *SqlServer2008Dialect* and so on.

What if you wrote some dialect with class name *MyOracleDialect"? It wouldn't match
the dialect string *"Oracle". *We also had that case covered by also using *ServerType*
property of dialect classes:

```cs
public class MyOracleDialect : IDialect 
{
    // ...

    public string ServerType 
    { 
        get { return "Oracle"; }
    }

}
```

So, for a dialect type to match an expression, its class name or ServerType property 
must start with the dialect name specified in the expression attribute.

# Specifying Multiple Database Types in One Expression

Remember that Sql Server versions before 2012 didn't have *CONCAT* operator? 
How would we handle that, like this?

```cs
[Expression("CONCAT(T0.[FirstName], CONCAT(' ', T0.[LastName]))")]
[Expression("(T0.FirstName + ' ' + T0.LastName)", Dialect = "SqlServer2000")]
[Expression("(T0.FirstName + ' ' + T0.LastName)", Dialect = "SqlServer2005")]
[Expression("(T0.FirstName + ' ' + T0.LastName)", Dialect = "SqlServer2008")]
[Expression("(T0.FirstName || ' ' || T0.LastName)", Dialect = "Sqlite")]
public String FullName
{
    get { return Fields.FullName[this]; }
    set { Fields.FullName[this] = value; }
}
```

But no, three lines with same expression doesn't feel nice. Luckily,
*Dialect* parameter can accept multiple dialects separated by comma:

```cs
[Expression("CONCAT(T0.[FirstName], CONCAT(' ', T0.[LastName]))")]
[Expression("(T0.FirstName + ' ' + T0.LastName)", 
    Dialect = "SqlServer2000, SqlServer2005, SqlServer2008")]
[Expression("(T0.FirstName || ' ' || T0.LastName)", Dialect = "Sqlite")]
public String FullName
{
    get { return Fields.FullName[this]; }
    set { Fields.FullName[this] = value; }
}
```

# A Tricky Method

Well, another trick would be to use it like this:

```cs
[Expression("CONCAT(T0.[FirstName], CONCAT(' ', T0.[LastName]))")]
[Expression("(T0.FirstName + ' ' + T0.LastName)", Dialect = "SqlServer200")]
[Expression("(T0.FirstName || ' ' || T0.LastName)", Dialect = "Sqlite")]
public String FullName
{
    get { return Fields.FullName[this]; }
    set { Fields.FullName[this] = value; }
}
```
SqlServer2000, SqlServer2005 and SqlServer2012 all start with "SqlServer200" right?

> I don't recommend relying on *starts with* functionality, it might be error prone,
and we may decide to change it later.

# A Better Trick

All SqlServer versions support *"+"* operator, so we would also write it like this:

```cs
[Expression("CONCAT(T0.[FirstName], CONCAT(' ', T0.[LastName]))")]
[Expression("(T0.FirstName + ' ' + T0.LastName)", Dialect = "SqlServer")]
[Expression("(T0.FirstName || ' ' || T0.LastName)", Dialect = "Sqlite")]
public String FullName
{
    get { return Fields.FullName[this]; }
    set { Fields.FullName[this] = value; }
}
```

# When Multiple Expressions Match

If multiple expressions match, one with *longest* dialect string will be used.

```cs
[Expression("CONCAT(T0.[FirstName], CONCAT(' ', T0.[LastName]))", "SqlServer")]
[Expression("(T0.FirstName || ' ' || T0.LastName)", Dialect = "SqlServer200")]
```

For example, SqlServer2000Dialect would match both of these expressions, but as
second expression has a longer *Dialect* string, it will be used.

# When Expression Is Determined

Expression for a field is determined at application start, and once it is set, 
it is *fixed*. Thus, you can't dynamically change it at runtime by changing dialect 
for a connection later.

Make sure you set correct dialect for connections in *SiteInitialization.cs* or
*web.config* method.

# Recommendations

We suggest to use this as a last resort, only when extremely required.

Make sure you write *the most generic/common* expression first, e.g. the one without a
*Dialect* string, and always have one such as a fallback expression.

Also don't depend too much on *starts with* functionality. We might decide to change 
this logic later. Try to use exact matches on *ServerType* and dialect class name.