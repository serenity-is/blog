---
title: Serenity 2.8.0 Upgrade Guide
description: 2.8.0 is an important step towards .NET Core support. This is the first version that ships NuGet packages with NetStandard1.6 targeted assemblies. Asp.Net Core port for Serene is currently a work in progress, but we are not there yet. So, this is an upgrade guide for existing users with ASP.NET MVC/.NET Framework 4.5.
author: Volkan Ceylan
---

Personally, i'm not a fan of Node/NPM/Bower etc, but sometimes you can't resist the change. 

For client side development they became something like standard, most javascript libraries are available in NPM/Bower, and these tools are already integrated into Visual Studio. 

NPM is also the recommended way to distribute / acquire TypeScript definition files in the form of `npm install @types/jquery` etc.

> Actually Bower, which is just another NPM package for client side libraries, looks like being deprecated and replaced by NPM itself.

So we are gonna start by using NPM for TypeScript definitions, instead of xyz.TypeScript.DefinitelyTyped NuGet packages.

> Another reason i have to do this is, content files in NuGet packages are no longer supported for ASP.NET Core project.json based projects, so we can't distribute .d.ts files with them.

After updating Serenity as usual, you'll have many TypeScript errors.

* First, start by ensuring that you have a `Serenity.CoreLib.d.ts` file under `scripts\typings\serenity`. This is the new location for this file.

* Then delete the old one at `scripts\serenity\Serenity.CoreLib.d.ts` (if you have any).

Now most of errors should be gone. When you open new `Serenity.CoreLib.d.ts` file or build your project you'll have a few errors like below:

```
Cannot find type definition file for 'jquery'.
Cannot find type definition file for 'jquery.validation'.
Cannot find type definition file for 'jqueryui'.
```

To resolve these, we have to install them with NPM.

First create a `package.json` file under YourApplication.Web project (where tsconfig.json and web.config resides).

Here is a sample one:

```json
{
  "name": "serene.web",
  "version": "1.0.0",
  "description": "My Application Description",
  "main": "index.js",
  "dependencies": {
    "@types/jquery": "^2.0.39",
    "@types/jquery.blockui": "0.0.27",
    "@types/jquery.cookie": "^1.4.28",
    "@types/jquery.validation": "^1.13.32",
    "@types/jqueryui": "^1.11.32",
    "@types/jspdf": "^1.1.31",
    "@types/sortablejs": "^1.3.31",
    "@types/toastr": "^2.1.32"
  },
  "devDependencies": {},
  "scripts": {
  },
  "author": "",
  "license": "ISC"
}
```

Replace `serene.web` with your app name.

Open a command prompt in the folder Serene.Web.csproj (YourProject.Web.csproj) exists.

> Right click on project name, open folder in explorer, open a command prompt from explorer File menu.

Type 

```cmd
C:\..\Serene.Web> npm
```

to make sure that you have Node/NPM installed. 

If not, install it from https://nodejs.org/en/download/ (LTS is recommended)

Again make sure that you are where web.config file resides (project root), then type:

```
C:\..\Serene.Web> npm install
```

You should now have a `node_modules` folder with a `@types` folder inside, under current directory.

> If you are using git, make sure you add `node_modules/` to `.gitignore` file.

Open package manager console and uninstall following packages:

```ps
Uninstall-Package toastr.TypeScript.DefinitelyTyped
Uninstall-Package sortablejs.TypeScript.DefinitelyTyped
Uninstall-Package jspdf.TypeScript.DefinitelyTyped
Uninstall-Package jquery.blockUI.TypeScript.DefinitelyTyped
Uninstall-Package jquery.cookie.TypeScript.DefinitelyTyped
Uninstall-Package jqueryui.TypeScript.DefinitelyTyped
Uninstall-Package jquery.validation.TypeScript.DefinitelyTyped
Uninstall-Package jquery.TypeScript.DefinitelyTyped
```

Now your project should build without any errors.

> Next step will be to move jQuery, jQuery-UI etc scripts to NPM but it is for another version.