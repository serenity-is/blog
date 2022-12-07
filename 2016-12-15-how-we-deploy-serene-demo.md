---
title: How Do We Deploy Serene Demo?
description: You may already know that we don't like hiding good things from you, thus we release often. If we didn't make use of some sort of auto deployment, it would be a real pain to publish Serenity NuGet packages, update Serene in VSGallery, and keep the live demo at https://demo.serenity.is up to date. Let's see how our basic Powershell script, combined with a GitHub webhook does the last one.
author: Volkan Ceylan
---

## Serene Demo Directory Structure

Serenity applications are x-copy deployable, as it should be with any ASP.NET MVC application. That means it shouldn't require anything other than setting up an IIS site / app and changing some settings like connection strings in web.config.

Serenity Demo is currently an application under **serenity.is** IIS root site.

Here is how its folder structure looks like:

```txt
C:\Sites
	SereneDemo\
		build\
			publish.ps1
			NuGet.exe
			NuGet.config
		clone\
			BuildTemplate.cmd
			Dev.Serene.sln
			Serene.sln
			Readme.md
			...
		web\
			bin\
			Content\
			Scripts\
			Views\
			favicon.ico
			Global.asax
			Web.config
```

I've an habit to not put my web sites under inetpub\wwwroot, so i usually prefer a separate directory like C:\Sites or similar.

SereneDemo is one of few sites under C:\Sites folder.

**build** directory contains our deployment script that we'll talk about (*publish.ps1*) along with a copy of NuGet.exe.

**clone** directory is a Git repository that contains a working copy of https://github.com/volkanceylan/Serene.git. I did clone it using Git Extensions (my preferred Git UI), but you may use any UI you like. Just make sure you have Git installed in your server.

Finally, **web** folder contains actual files for our application, Serene Demo that is published at https://demo.serenity.is/. This is the folder that is mapped in IIS to **demo** application.

You only need to have a **Web.config** file under this directory, before running the publish script.

Let's examine our build script.

## Console I/O Related Helpers

I must first admit that i don't think myself as a Powershell guru. I just learned what i need so far and pretty sure that any Powershell you'll see here could be done in a much better way.

```ps
param (
    [int]$auto = 0
)

function pressAnyKeyToExit() {
	if ($auto -eq 0) {
		try {
			Read-Host 'Press any key to exit...' | Out-Null
		} 
		catch {
		}
	}
}

function writeCaption($status) {
    echo ''
    write-host ('######## ' + $status + ' ########') -foregroundcolor "green"
    echo ''
}

function exitWithError($message, $exitcode = -1) {
    write-host ''
    write-host 'ERROR: ' $message -foregroundcolor "red"
    write-host ''
    pressAnyKeyToExit
    Exit $exitCode
}


```

Our script starts with defining its command line arguments, or the only parameter, **$auto**. 

It will control if *pressAnyKeyToExit()* method will actually wait for user to press some key. It will be helpful for times when this script is run from an automated task. If i run it manually, i would like it actually wait, so that i can see what's going wrong before the console window is closed.

*writeCaption* is a very simple helper to print a noticeable heading to console.

*exitWithError()* method prints a message to screen with red color, waits for user to press a key then exits the script with an Exit Code (-1 by default).

Here comes our first method that will make use of *exitWithError*.

```
function testedCd($path) {
    try {
      Set-Location $path -ErrorAction Stop
    }
    catch {
        exitWithError('Cant cd to ' + $path + ' directory')   
    }
}
```

This method tries to CD to a given path, and exits the application with an error message if can't. There are so many times that i remember, when i tried to CD into a non existent directory and kept moving like i'm in it, so i use this method to be certain.

## Determining the Location of *MsBuild*

My quick and not so elegant method of locating the **msbuild.exe**:

```ps
$dotnetdir = $env:SystemRoot + '\Microsoft.NET\Framework64\v4.0.30319'
if (-not (Test-Path $dotnetdir)) {
    $dotnetdir = $env:SystemRoot + '\Microsoft.NET\Framework\v4.0.30319'
}
$prgx86 = ${env:ProgramFiles(x86)};
$msbuild = $prgx86 + '\MSBuild\14.0\Bin\amd64\MSBuild.exe';
if (-not (Test-Path $msbuild)) {
    $msbuild = $prgx86 + '\MSBuild\13.0\Bin\amd64\MSBuild.exe';
    if (-not (Test-Path $msbuild)) {
        $msbuild = $prgx86 + '\MSBuild\12.0\Bin\amd64\MSBuild.exe';
        if (-not (Test-Path $msbuild)) {
            $msbuild = $dotnetdir + '\MSBuild.exe';
        }
    }
}
```

.NET framework might be under different directories for 32 and 64 bit systems, so i first determine its path, giving priority to 64 bit.

Next step is to get location for *Program Files (x86)* system folder.

Starting with Visual Studio 2013 (if my memory doesn't fail me), *msbuild* is placed under *Program Files (x86)\MSBuild* instead of .NET framework directory.

So i try to find the latest version MSBuild possible. As a last resort, i fallback to one under .NET directory.

## Building the Serene Project

```ps
$builddir = Split-Path -parent $MyInvocation.MyCommand.Definition
$nuget = $builddir + '\NuGet.exe'
$rootdir = (Split-Path -parent $builddir)
$solutiondir = $rootdir + '\clone';

function buildProject($sln) {

    writeCaption('Restoring NuGet Packages for Solution')
    & $nuget restore $sln
    
    writeCaption('Building Project/Solution in Release Mode')   
    & $msbuild $sln /m:8 /t:rebuild /p:Configuration=Release /v:Minimal
    if ($LASTEXITCODE -ne 0) {
        echo $LASTEXITCODE
        exitWithError('msbuild returned with ' + $LASTEXITCODE + ' exit code!')
    }
}
```

I don't like hardcoding any paths, so i start by finding the path of our running script (*publish.ps1*) by making use of *$MyInvocation.MyCommand.Definition*.

NuGet.exe should be also under same build directory.

Our script is under *SereneDemo\build*, so parent of that directory is the root folder for *SereneDemo*.

Serene.csproj resides under *SereneDemo\clone* so *$solutiondir* will hold the location of that directory.

Our helper method *buildProject* will start by restoring packages for given solution / project.

We then invoke *MsBuild* with *Release* configuration and *Rebuild* mode.

If *MsBuild* results in any compile errors, we shouldn't continue deployment.

We'll call *buildProject* method soon.


## Pulling Latest Changes From Serene GitHub Repository

```ps
try {
	testedCd $solutiondir
	& $git fetch --all
	& $git reset --hard origin/master
	buildProject $solutiondir\Serene.sln

	//...
} finally {
	testedCd $builddir
}
```

We start by test CD to solution dir (clone folder) residing under our working copy of GitHub Serene repository.

Invoking *git fetch* and *git reset* commands should be enough to fetch and checkout latest commit on master branch.

Next step is to invoke our *buildProject* method passing *Serene.Web.csproj* location.

## Copying Changes to Deployment Folder

After building the project, i'll use robocopy to copy required files from *C:\Sites\SereneDemo\clone* directory to *C:\Sites\SereneDemo\web*.

Robocopy allows specifiying file and directory ignore patterns, so we can exclude files like *.cs*, *.ts*, *.suo* etc. while deploying.

Unfortunately, i couldn't find a way to ignore the Web.config at root directory, while including others under *Views*, *Modules* etc, so i had to use a dirty workaround:

```ps
function copyFilesToDeployment($src, $deployment) {
    writeCaption('Copying Changes To Deployment Folder')
    testedCd $src
    robocopy /s /MT:4 /np $src $deployment /xd App_Data obj "Service References" /xf *.git *.cs *.ts *.suo *.vspcc *.csproj *.csproj.user *.orig *.gitignore packages.config *.bak Web.Debug.Config Web.Release.Config *.datasource *.svcinfo ($src + '\web.config')
    testedCd $deployment
}

if (Test-Path ($solutiondir + '\Serene\Serene.Web\Renamed.Web.config')) {
	del ($solutiondir + '\Serene\Serene.Web\Renamed.Web.config') -force
}
ren ($solutiondir + '\Serene\Serene.Web\Web.config') Renamed.Web.config
try {
	copyFilesToDeployment ($solutiondir + '\Serene\Serene.Web\') ($rootdir + '\web\')
	del ($rootdir + '\web\Renamed.Web.config') -force
} finally {
	ren ($solutiondir + '\Serene\Serene.Web\Renamed.Web.config') Web.config
}
```

This simply renames *Web.config* at our *clone* directory to *Renamed.Web.config* temporarily, so that we won't override the Web.config under *Web* folder while copying.

## Running the Script

Now it's time to run *publish.ps1* with Powershell and see if we have any errors.

You might need to enable script execution for Powershell scripts at first run.

You need tools like .NET framework, Git etc. installed and any third party packages your project might require.

## Automating the Deployment Using a GitHub WebHook

GitHub allows you to call some web address when some change like *push to master* occurs at your repository. So i simply set up such an hook, and called my Powershell script when it happens.




