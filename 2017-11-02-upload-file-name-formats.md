---
title: Upload File Name Formats
description: Serenity has a unique upload system that is pretty flexible. Although it works as is, without requiring much configuration, you might want to customize it sometimes. One of the areas you might want to learn about is generated file names on disk and the format specifiers that controls them. A new feature added in recent versions is ability to use original file name on disk.
author: Volkan Ceylan
---

## Where Files Are Stored

Uploaded files goes under *App_Data/upload* folder.

A setting in web.config controls the upload path:

```xml
<add key="UploadSettings" value="{ Path: '~\\App_Data\\upload\\' }" />
```

## Default File Name Format

Unless you change the default format, Serenity generates unique filenames on disk like *ProductImage/0000/00000002_4qqvi6xw25suq.png* under *AppData/upload* when you upload a file with name *MyProductImage.png* for example. We'll dissect this naming scheme soon.

## Where File Name is Stored

When you add a [(File/Image)UploadEditor], e.g. single file upload editor to a string field, Serenity automatically stores file name on disk, e.g. *ProductImage/0000/00000002_4qqvi6xw25suq.png* in that field.

## How Can I Save Original Name When Using Single File Upload?

You may set OriginalNameProperty to a different field to store original file name along with disk file name, e.g.

```cs
[ImageUploadEditor(OriginalNameProperty = "MyImageName")]
public string MyImageFile { get; set; }

public string MyImageName { get; set; }
```

After upload, *MyImageFile* field will hold the file name on disk (*ProductImage/0000/00000002_4qqvi6xw25suq.png*) and *MyImageName* field will store the original file name of uploaded file (*MyProductImage.png*).

## The Case With Multiple Upload Editors

When you use a [Multiple(File/Image)UploadEditor] it also stores file names in a string field, but this time in JSON format. You also won't need the original name property as it is also saved into same field.

```cs
[MultipleImageUploadEditor]
public string MyImageFiles { get; set;}
```

After upload *MyImageFiles* field will have content like this:

```js
[
    {
        "FileName": "ProductImage/0000/00000002_4qqvi6xw25suq.png",
        "OriginalName": "FirstImage.png"
    },
    {
        "FileName": "ProductImage/0000/00000002_ax4qi1f435tst.png",
        "OriginalName": "SecondImage.png"
    }
]
```

## Dissecting File Naming Scheme

Generated file name (e.g. *ProductImage/0000/00000002_4qqvi6xw25suq.png*) on disk has several parts:

- A target field specific folder (*ProductImage*)
- A grouping folder based on record ID (*0000*)
- Record ID itself (*0000002*)
- Random unique suffix (*4qqvi6xw25suq*)

## Why We Use Such a Naming Scheme?

There are several reasons this is the default behavior:

* This prevents clashes when files with same file names are uploaded for different records.
* Files can be identified by their related record IDs
* A folder won't have more than 1000 files thanks to *grouping*. Most file systems doesn't perform well when a directory has hundreds of thousands of files.
* Even if a file with same name but different contents is uploaded, you won't have cache problems as file name prefix (*00000002_4qqvi6xw25suq*) changes after every upload.

## File Name Format

The default file name format is a .NET string format specifier:

```
ProductImage/{1:0000}/{0:00000000}_{2}
```

If you look at the *ProductImage* field of *ProductRow*:

```cs
[ImageUploadEditor(FilenameFormat = "ProductImage/~", CopyToHistory = true)]
public String ProductImage
{
    get { return Fields.ProductImage[this]; }
    set { Fields.ProductImage[this] = value; }
}
```

You'll see that it only has "`~`" character after *ProductImage/*. This character corresponds to *{1:0000}/{0:00000000}_{2}* which is the default naming format.

> If you don't specify any FilenameFormat, the folder name is generated based on your row class name and property name.

## Format Specifiers of the Default File Name Format

- **{0}** is the record ID.
- **{1}** is the record ID divided by 1000. If record ID is not a numeric value, its the first two characters of record ID.
- **{2}** is a random string of 13 characters (8 bytes of binary array encoded in Base32)
- **{3}** is current date/time. You may use it to add file upload date / time to generated file name as you like.

## Ability to Use Original Name (introduced in Serenity 2.10.2)

New format specifier (**{4}**) is recently added in Serenity which corresponds to original file name. Thus, you now have the option to use original name on disk as is.

But when a file with same name already exists on disk, Serenity will automatically add a suffix.

So for example if a user uploads *SomeFile.docx* and another also uploads *SomeFile.docx*, the second one will have name of *SomeFile (2).docx" on disk.

## Referencing Other Row Fields in File Names (introduced in 2.4.4)

Sometimes you might want to refer some other field in same row while generating file name.

It is possible by using pipes, e.g. `|OtherFieldName|` in file name formats.

```cs
[ImageUploadEditor(FilenameFormat = "|SupplierId|/ProductImage/{4}")]
public String ProductImage
{
    get { return Fields.ProductImage[this]; }
    set { Fields.ProductImage[this] = value; }
}
```

Avoid using foreign fields as they are not available yet in new records.

## Conclusion

Serenity has a very dynamic file upload name generation system that should handle most requirements.

It might not be possible to provide support for every custom need out there but Serenity also provides you flexibility of writing your own custom file upload behavior.