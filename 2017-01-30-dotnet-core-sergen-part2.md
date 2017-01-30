---
title: Sergen for .NET Core - Part 2
description: We continue where we left on Part 1 about determining primary keys, identity fields, foreign keys and other table column information on a bunch of databases. 
author: Volkan Ceylan
---

## Primary Key Columns

**Firebird:**:

```cs
public IEnumerable<string> GetPrimaryKeyFields(IDbConnection connection, 
     string schema, string table)
{
    return connection.Query<string>(@"
        SELECT ISGMT.RDB$FIELD_NAME FROM
        RDB$RELATION_CONSTRAINTS rc
        INNER JOIN RDB$INDEX_SEGMENTS ISGMT ON rc.RDB$INDEX_NAME = ISGMT.RDB$INDEX_NAME
        WHERE CAST(RC.RDB$RELATION_NAME AS VARCHAR(40)) = @tbl 
            AND RC.RDB$CONSTRAINT_TYPE = 'PRIMARY KEY'
        ORDER BY ISGMT.RDB$FIELD_POSITION", new { tbl = table })
            .Select(StringHelper.TrimToNull);
}
```

We use some system tables, and index information to determine primary keys in firebird. In Firebird, 
system string columns are CHAR, e.g. right padded with space to a certain length, so we need to
right trim anything we receive.

**MySql:**:

```cs
public IEnumerable<string> GetPrimaryKeyFields(IDbConnection connection,
    string schema, string table)
{
    return connection.Query<string>(@"
            SELECT COLUMN_NAME  
            FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS AS tc  
            INNER JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE AS ku  
            USING(constraint_name,table_schema,table_name)
            WHERE tc.CONSTRAINT_TYPE = 'PRIMARY KEY'  
            AND tc.CONSTRAINT_NAME = ku.CONSTRAINT_NAME  
            AND ku.TABLE_SCHEMA = Database()  
            AND ku.TABLE_NAME = @tbl  
            ORDER BY ku.ORDINAL_POSITION",
        new
        {
            tbl = table
        });
}
```

MySql implements information schema, close to standard, so its kinda easier.

**Oracle:**:

```cs
public IEnumerable<string> GetPrimaryKeyFields(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<string>(@"
            SELECT cols.column_name
            FROM all_constraints cons, all_cons_columns cols
            WHERE cols.table_name = :tbl
            AND cons.constraint_type = 'P'
            AND cons.constraint_name = cols.constraint_name
            AND cons.owner = :sch
            ORDER BY cols.position;",
        new
        {
            sch = schema,
            tbl = table
        });
}

```

I couldn't test this, so if anyone with Oracle installed can confirm it works, i'd appreciate.

**Postgres:**:

```cs
public IEnumerable<string> GetPrimaryKeyFields(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<string>(
        "SELECT pg_attribute.attname " +
        "FROM pg_index, pg_class, pg_attribute, pg_namespace " +
        "WHERE pg_class.oid = '\"" + table + "\"'::regclass " +
        "AND indrelid = pg_class.oid " +
        "AND nspname = '" + schema + "' " +
        "AND pg_class.relnamespace = pg_namespace.oid " +
        "AND pg_attribute.attrelid = pg_class.oid " +
        "AND pg_attribute.attnum = any(pg_index.indkey) " +
        "AND indisprimary");
}
```

**Sqlite**:

```cs
public IEnumerable<string> GetPrimaryKeyFields(IDbConnection connection, string schema, string table)
{
    return connection.Query("PRAGMA table_info([" + table + "])")
        .Where(x => (int)x.pk > 0)
        .OrderBy(x => (int)x.pk)
        .Select(x => (string)x.name);
}
```

Sqlite looks like easiest thanks to its PRAGMA directive.

**SqlServer**:

```cs
public IEnumerable<string> GetPrimaryKeyFields(IDbConnection connection, string schema, string table)
{
    return connection.Query<string>(
            "SELECT COLUMN_NAME " +
            "FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS AS tc " +
            "INNER JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE AS ku " +
            "ON tc.CONSTRAINT_TYPE = 'PRIMARY KEY' " +
            "AND tc.CONSTRAINT_NAME = ku.CONSTRAINT_NAME " +
            "AND ku.TABLE_SCHEMA = @schema " +
            "AND ku.TABLE_NAME = @table " +
            "ORDER BY ku.ORDINAL_POSITION",
        new
        {
            schema = schema,
            table = table
        });
}
```

Sql Server implements INFORMATION_SCHEMA just like MySql, so their queries are pretty similar.

## Identity Columns

This one is a bit trickier than detecting primary keys, as each database has a different method
of generating identity columns (e.g. auto incrementing integer values).

Even though almost all of them supports some kind of *identity*, it is implemented with varying
methods and it is not easy to find which field is an identity.


**Firebird:**:

It's almost impossible to determine identity columns in Firebird. There is something called *GENERATOR*,
which lets you generate auto incrementing numbers, but you have to write a separate trigger to
set a columns value using that generator.

Thus, to determine which column is an identity column, you would have to parse trigger code, which
would be hard, and not always possible.

So, i had to use some trick. I checked what name FluentMigrator Firebird provider generates
for these generators behind the scenes.

For a table named USERS, with column USERID, the generated sequence name is

`GEN_USERS_USERID`

Hoping that users will use FluentMigrator to create tables, or at least use a similar naming scheme,
i tried to use this naming schema to find the identity column.

One problem is if the table or column name is long, as Firebird has a limit on object names 
(32 chars) generator name might be trimmed. For table MeetingAgendas with MeetingAgendaId
identity column, generator name would be:

`GEN_MEETINGAGENDAS_MEETINGAGEN`

So, we have to handle that case as well. 

In worst case, where there is no such named generator, i'll assume first primary key as
the identity.

Here is the final code:

```cs
public IEnumerable<string> GetIdentityFields(IDbConnection connection, 
    string schema, string table)
{
    var match = connection.Query<string>(@"
            SELECT RDB$GENERATOR_NAME
            FROM RDB$GENERATORS
            WHERE RDB$SYSTEM_FLAG = 0 AND RDB$GENERATOR_NAME LIKE @genprefix",
        new
        {
            genprefix = "GEN_" + table + "_%"
        })
        .Select(x => x.Substring(("GEN_" + table + "_").Length))
        .FirstOrDefault()
        .TrimToNull();

    if (match == null)
    {
        var primaryKeys = this.GetPrimaryKeyFields(connection, schema, table);
        if (primaryKeys.Count() == 1)
            return primaryKeys;

        return new List<string>();
    }

    return connection.Query<string>(@"
            SELECT RDB$FIELD_NAME
            FROM RDB$RELATION_FIELDS
            WHERE RDB$RELATION_NAME = @tbl
            AND RDB$FIELD_NAME LIKE @match
            ORDER BY RDB$FIELD_POSITION",
        new
        {
            tbl = table,
            match = match + "%"
        })
        .Take(1)
        .Select(StringHelper.TrimToNull);
}
```

**MySql:**:

```cs
public IEnumerable<string> GetIdentityFields(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<string>(@"
            SELECT COLUMN_NAME FROM information_schema.COLUMNS 
            WHERE TABLE_SCHEMA = Database()
            AND table_name = @tbl
            AND EXTRA = 'auto_increment'", 
        new
        {
            tbl = table
        });
}
```

MySql is much easier, as it returns `auto_increment` value for identity columns on EXTRA field.

**Oracle:**:

```cs
public IEnumerable<string> GetIdentityFields(IDbConnection connection, 
    string schema, string table)
{
    return new List<string>();
}
```

AFAIK, Oracle supports identity columns in recent versions, but i don't know if there is an easy
way to determine them. Help, anyone?

**Postgres:**:

```cs
public IEnumerable<string> GetIdentityFields(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<string>(@"
        SELECT column_name, column_default 
        FROM information_schema.COLUMNS 
        WHERE TABLE_SCHEMA = @sma AND TABLE_NAME = @tbl 
        AND column_default like 'nextval(%'", new
    {
        sma = schema,
        tbl = table
    });
}
```

Postgres also has *generators* or *sequences* just like Firebird, but at least it is kinda
possible to determine where these generators are used, by checking default value information
of columns. If `nextval` function is used in default value, that is probably an identity 
column. 

**Sqlite**:

```cs
public IEnumerable<string> GetIdentityFields(IDbConnection connection, string schema, string table)
{
    var fields = connection.Query("PRAGMA table_info([" + table + "])")
        .Where(x => (int)x.pk > 0);

    if (fields.Count() == 1 &&
        (string)fields.First().type == "INTEGER")
    {
        return new List<string> { (string)fields.First().name };
    };

    return new List<string> { "ROWID" };
}
```

Sqlite has an auto incrementing ROWID column by default for any table. You may *rename* it to something
else by adding an *INTEGER PRIMARY KEY* column.

**SqlServer**:

```cs
public IEnumerable<string> GetIdentityFields(IDbConnection connection,
    string schema, string table)
{
    return connection.Query<string>(@"
            SELECT COLUMN_NAME
            FROM INFORMATION_SCHEMA.COLUMNS
            WHERE TABLE_SCHEMA = @schema AND TABLE_NAME = @table
            AND COLUMNPROPERTY(object_id(TABLE_SCHEMA + '.' + TABLE_NAME), 
                COLUMN_NAME, 'IsIdentity') = 1",
        new
        {
            schema = schema,
            table = table
        });
}
```

Sql Server has an identity column type out of the box, so its simpler to detect which by
using INFORMATION_SCHEMA.


## Foreign keys

Sergen has to know about foreign keys to identify relations between tables, and bring in first level
view fields from foreign tables.

We only need name of foreign key for grouping, and list of matched fields.

**Firebird:**:

```cs
public IEnumerable<ForeignKeyInfo> GetForeignKeys(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<ForeignKeyInfo>(@"
        SELECT
            PK.RDB$RELATION_NAME as PKTable,
            ISP.RDB$FIELD_NAME as PKColumn,
            FK.RDB$CONSTRAINT_NAME as FKName
        FROM
            RDB$RELATION_CONSTRAINTS PK, 
            RDB$RELATION_CONSTRAINTS FK, 
            RDB$REF_CONSTRAINTS RC, 
            RDB$INDEX_SEGMENTS ISP, 
            RDB$INDEX_SEGMENTS ISF
            WHERE FK.RDB$RELATION_NAME = @tbl
            AND FK.RDB$CONSTRAINT_NAME = RC.RDB$CONSTRAINT_NAME
            AND PK.RDB$CONSTRAINT_NAME = RC.RDB$CONST_NAME_UQ
            AND ISP.RDB$INDEX_NAME = PK.RDB$INDEX_NAME
            AND ISF.RDB$INDEX_NAME = FK.RDB$INDEX_NAME
            AND ISP.RDB$FIELD_POSITION = ISF.RDB$FIELD_POSITION
            ORDER BY ISP.RDB$FIELD_POSITION", new
    {
        tbl = table
    }).Select(x =>
    {
        x.FKName = x.FKName.TrimToNull();
        x.FKColumn = x.FKColumn.TrimToNull();
        x.PKColumn = x.PKColumn.TrimToNull();
        x.PKTable = x.PKTable.TrimToNull();
        return x;
    });
}
```

**MySql:**:

```cs
public IEnumerable<ForeignKeyInfo> GetForeignKeys(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<ForeignKeyInfo>(@"
        SELECT 
            i.CONSTRAINT_NAME FKName,
            k.COLUMN_NAME FKColumn, 
            CASE WHEN k.REFERENCED_TABLE_SCHEMA = Database() THEN NULL 
                ELSE k.REFERENCED_TABLE_SCHEMA END PKSchema, 
            k.REFERENCED_TABLE_NAME PKTable,
            k.REFERENCED_COLUMN_NAME PKColumn
        FROM information_schema.TABLE_CONSTRAINTS i
        LEFT JOIN information_schema.KEY_COLUMN_USAGE k ON i.CONSTRAINT_NAME = k.CONSTRAINT_NAME
        WHERE i.CONSTRAINT_TYPE = 'FOREIGN KEY'
        AND i.TABLE_SCHEMA = Database()
        AND i.TABLE_NAME = @tbl", new
    {
        tbl = table
    });
}
```

**Oracle:**:

```cs
public IEnumerable<ForeignKeyInfo> GetForeignKeys(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<ForeignKeyInfo>(@"
        SELECT 
            a.constraint_name FKName,                     
            a.column_name FKColumn,
            c.r_owner PKSchema,
            c_pk.table_name PKTable,
            uc.column_name PKColumn
        FROM all_cons_columns a
        JOIN all_constraints c ON a.owner = c.owner AND a.constraint_name = c.constraint_name
        JOIN all_constraints c_pk ON c.r_owner = c_pk.owner AND c.r_constraint_name = c_pk.constraint_name
        JOIN user_cons_columns uc ON uc.constraint_name = c.r_constraint_name
        WHERE c.constraint_type = 'R' 
            AND a.table_name = :tbl
            AND c.r_owner = :sma", new
    {
        sma = schema,
        tbl = table
    });
}
```

I couldn't test this one as usual for Oracle, can somebody please?

**Postgres:**:

```cs
public IEnumerable<ForeignKeyInfo> GetForeignKeys(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<ForeignKeyInfo>(@"
        SELECT
            o.conname AS FKName,
            (SELECT a.attname FROM pg_attribute a WHERE 
                a.attrelid = m.oid AND a.attnum = o.conkey[1] 
                AND a.attisdropped = false) AS FKColumn,
            (SELECT nspname FROM pg_namespace WHERE oid=f.relnamespace) AS PKSchema,
            f.relname AS PKTable,
            (SELECT a.attname FROM pg_attribute a WHERE a.attrelid = f.oid 
                AND a.attnum = o.confkey[1] AND a.attisdropped = false) AS PKColumn
        FROM
            pg_constraint o LEFT JOIN pg_class c ON c.oid = o.conrelid
            LEFT JOIN pg_class f ON f.oid = o.confrelid LEFT JOIN pg_class m ON m.oid = o.conrelid
        WHERE
            o.contype = 'f' AND o.conrelid IN (SELECT oid FROM pg_class c WHERE c.relkind = 'r')
            AND (SELECT nspname FROM pg_namespace WHERE oid=m.relnamespace) = @sma
            AND m.relname = @tbl", new
    {
        sma = schema,
        tbl = table
    });
}
```

**Sqlite**:

```cs
public IEnumerable<ForeignKeyInfo> GetForeignKeys(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query("PRAGMA foreign_key_list([" + table + "])")
        .Select(x => new ForeignKeyInfo
        {
            FKName = x.id.ToString(),
            FKColumn = x.from,
            PKTable = x.table,
            PKColumn = x.to
        });
}
```

I'm not sure if Sqlite enforces FK constraints, but at least it has some
simple way to list them.

**SqlServer**:

```cs
public IEnumerable<ForeignKeyInfo> GetForeignKeys(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<ForeignKeyInfo>(@"
        SELECT
            fk.CONSTRAINT_NAME FKName,
            fk.COLUMN_NAME FKColumn,
            pk.TABLE_SCHEMA PKSchema,
            pk.TABLE_NAME PKTable,
            pk.COLUMN_NAME PKColumn
        FROM 
            INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS c,
            INFORMATION_SCHEMA.CONSTRAINT_COLUMN_USAGE fk,
            INFORMATION_SCHEMA.KEY_COLUMN_USAGE pk 
        WHERE
            fk.TABLE_SCHEMA = @sma
            AND fk.TABLE_NAME = @tbl
            AND fk.CONSTRAINT_SCHEMA = c.CONSTRAINT_SCHEMA
            AND fk.CONSTRAINT_NAME = c.CONSTRAINT_NAME
            AND pk.CONSTRAINT_SCHEMA = c.UNIQUE_CONSTRAINT_SCHEMA
            AND pk.CONSTRAINT_NAME = c.UNIQUE_CONSTRAINT_NAME", new
    {
        sma = schema,
        tbl = table
    });
}
```

## List of Columns and Their Type Information

**Firebird:**:

```cs
public IEnumerable<FieldInfo> GetFieldInfos(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query(@"
        SELECT
            rfr.rdb$field_name AS FIELD_NAME,
            fld.rdb$field_type AS FIELD_TYPE,
            fld.rdb$field_sub_type AS FIELD_SUB_TYPE,
            CAST(fld.rdb$field_length AS integer) AS SIZE,
            CAST(fld.rdb$field_precision AS integer) AS NUMERIC_PRECISION,
            CAST(fld.rdb$field_scale AS integer) AS NUMERIC_SCALE,
            CAST(fld.rdb$character_length AS integer) AS CHARMAXLENGTH,
            coalesce(fld.rdb$null_flag, rfr.rdb$null_flag) AS COLUMN_NULLABLE
        FROM rdb$relation_fields rfr
            LEFT JOIN rdb$fields fld ON rfr.rdb$field_source = fld.rdb$field_name
        WHERE
            rfr.rdb$relation_name = @tbl
        ORDER BY 
            rfr.rdb$field_position",  new
    {
        tbl = table   
    }).Select(src =>
    {
        var fi = new FieldInfo();
        fi.FieldName = ((string)src.FIELD_NAME).TrimToNull();
        var fieldType = src.FIELD_TYPE == null ? 0 : 
            Convert.ToInt32(src.FIELD_TYPE, CultureInfo.InvariantCulture);
        var fieldSubType = src.FIELD_SUB_TYPE == null ? 0 : 
            Convert.ToInt32(src.FIELD_SUB_TYPE, CultureInfo.InvariantCulture);
        var numericScale = src.NUMERIC_SCALE == null ? 0 : 
            Convert.ToInt32(src.NUMERIC_SCALE, CultureInfo.InvariantCulture);
        var numericPrecision = src.NUMERIC_PRECISION == null ? 0 : 
            Convert.ToInt32(src.NUMERIC_PRECISION, CultureInfo.InvariantCulture);
        var size = src.SIZE == null ? 0 : Convert.ToInt32(src.SIZE, CultureInfo.InvariantCulture);
        var sqlType = GetSqlTypeFromBlrType(fieldType, fieldSubType, size, numericScale);
        fi.DataType = sqlType;

        if (sqlType == "char" || sqlType == "varchar")
            fi.Size = src.CHARMAXLENGTH == null ? 0 : Convert.ToInt32(src.CHARMAXLENGTH);
        else if (sqlType == "varbinary" || sqlType == "text")
            fi.Size = 0;
        else if (sqlType == "decimal" || sqlType == "numeric")
        {
            fi.Size = numericPrecision;
            fi.Scale = -numericScale;
        }
        fi.IsNullable = src.COLUMN_NULLABLE == null;

        return fi;
    });
}
```

I had to reference code from Firebird.Data provider to extract column information properly.
Source code for GetSqlTypeFromBlrType can be found in Serenity repository.

**MySql:**:

```cs
public IEnumerable<FieldInfo> GetFieldInfos(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query(string.Format("SHOW FULL COLUMNS FROM `{0}`", table))
        .OrderBy(x => Convert.ToInt32(x.ORDINAL_POSITION))
        .Select(src =>
        {
            var fi = new FieldInfo();
            fi.FieldName = src.Field;
            fi.IsNullable = ((string)src.Null) != "NO";
            var dataType = (string)src.Type;
            var dx = dataType.IndexOf('(');
            if (dx >= 0)
            {
                var dxend = dataType.IndexOf(')', dx);
                var strlen = dataType.Substring(dx + 1, dxend - dx - 1);
                dataType = dataType.Substring(0, dx);
                var lower = dataType.ToLowerInvariant();
                if (lower == "char" || lower == "varchar")
                    fi.Size = int.Parse(strlen);
                else if (lower == "real" || lower == "decimal")
                {
                    var strparts = strlen.Split(',');
                    fi.Size = int.Parse(strparts[0]);
                    fi.Scale = int.Parse(strparts[1]);
                }
            }
            fi.DataType = dataType;
            fi.IsPrimaryKey = (string)src.Key == "PRI";
            fi.IsIdentity = (string)src.Extra == "auto_increment";
            return fi;
        });
}
```

Compared to Firebird, MySql is a bit easier.

**Oracle:**:

```cs
public IEnumerable<FieldInfo> GetFieldInfos(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<FieldInfo>(@"
        SELECT 
            c.column_name ""FieldName"",
            c.data_type ""DataType"",
            COALESCE(NULLIF(c.data_precision, 0), c.char_length) ""Size"",
            c.data_scale ""Scale"",
            CASE WHEN c.nullable = 'N' THEN 1 ELSE 0 END ""IsNullable"",
        FROM all_tab_columns c
        WHERE 
            c.owner = :sma
            AND c.table_name = :tbl
        ORDER BY c.column_id
    ", new
    {
        sma = schema,
        tbl = table
    });
}
```

Just another blindly written, untested code for Oracle.

**Postgres:**:

```cs
public IEnumerable<FieldInfo> GetFieldInfos(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<FieldInfo>(@"
        SELECT  
            column_name ""FieldName"",
            data_type ""DataType"",
            CASE WHEN is_nullable = 'NO' THEN 0 ELSE 1 END ""IsNullable"",
            CASE WHEN column_default LIKE 'nextval(%' THEN 1 ELSE 0 END ""IsIdentity"",
            COALESCE(character_maximum_length, CASE WHEN data_type = 'numeric' OR 
                data_type = 'decimal' THEN numeric_precision ELSE 0 END) ""Size"",
            numeric_scale ""Scale""
        FROM information_schema.COLUMNS
        WHERE table_schema = @sma and table_name = @tbl
        ORDER BY ordinal_position", new
    {
        sma = schema,
        tbl = table
    });
}
```

Postgres looks simpler than others.

**Sqlite**:

```cs
public IEnumerable<FieldInfo> GetFieldInfos(IDbConnection connection, string schema, string table)
{
    return connection.Query("PRAGMA table_info([" + table + "])")
        .Select(x => new FieldInfo
        {
            FieldName = x.name,
            DataType = x.type,
            IsNullable = Convert.ToInt32(x.notnull) != 1,
            IsPrimaryKey = Convert.ToInt32(x.pk) == 1
        });
}
```

Well, no one can beat Sqlite when it comes to simplicity but it misses some extra information
like scale, size etc. in return. For example, string columns has no character length limit.

**SqlServer**:

```cs
public IEnumerable<FieldInfo> GetFieldInfos(IDbConnection connection, 
    string schema, string table)
{
    return connection.Query<FieldInfo>(@"
        SELECT
            COLUMN_NAME [FieldName],
            CASE WHEN DATA_TYPE = 'timestamp' THEN 'rowversion' ELSE DATA_TYPE END [DataType],
            CASE WHEN IS_NULLABLE = 'NO' THEN 0 ELSE 1 END [IsNullable],
            COALESCE(CHARACTER_MAXIMUM_LENGTH, 
                CASE WHEN DATA_TYPE in ('decimal', 'money', 'numeric') 
                    THEN NUMERIC_PRECISION ELSE 0 END) [Size],
            NUMERIC_SCALE [Scale]
        FROM
            INFORMATION_SCHEMA.COLUMNS
        WHERE
            TABLE_SCHEMA = @sma 
            AND TABLE_NAME = @tbl
        ORDER BY
            ORDINAL_POSITION", new
    {
        sma = schema,
        tbl = table
    });
}
```

Sergen was initially designed for Sql Server, so put a few CASE statements here and there,
and you have a working code.

## Conclusion

It was a long post, with much code, and less explanations, but in the end we now have a Sergen
for .NET Core version that works with all these database types except Oracle, as it doesn't have
a .NET Core provider yet.

Now the code is more organized and it will be easier to add new providers in the future. 

If you spot some mistakes or have some improvements to suggest, let me know in comments.

Next step is to backport these providers to Sergen for Net45.