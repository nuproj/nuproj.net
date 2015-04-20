---
title: Documentation
layout: documentation
---

# Getting started

NuProj adds a new project type to Visual Studio that let's you build NuGet
packages.

It supports:

1. **Referencing other projects (such as C# and VB projects)**. This will cause
their project outputs to be packaged under the lib folder. NuProj takes care of
figuring out the right lib folder name, such as net45. It will also promote any
NuGet references to proper dependencies in the resulting .nupkg file. You can
also reference other NuProj projects which allows you to declare dependencies to
packages that are part of the same solution.

2. **Manually maintained content**. You can also add content directly to the
project. In that case, you'd simply use folders in the Visual Studio project to
control how the content is packaged. For example, in order to get an install
script, you'd create a tools folder and within that you'd add the install.ps1
file.

3. **Producing symbol packages**. You can easily create symbol packages.
Optionally, you can also include the sources. In this case, NuProj will use the
PDB information to find the source files and embed them in the resulting .nupkg
file.

4. **Build servers such as Visual Studio Online**. The build integration is also
available separately and can be checked in or
[restored as a NuGet package](#checked-in-toolkit).

## Creating a NuGet package

Create a new project in Visual Studio by invoking the menu command
**File | New | Project** or by pressing <kbd>Ctrl + Shift + N</kbd>. In the
**New Project** dialog, select the **NuGet Package** template from the **NuGet**
category. Click **OK**.

![New Project dialog](/images/NewProject.png)

## Adding automatic content

Let's say you want to package a class library, e.g. `ClassLibrary1`. In order
to automatically package the output of this class library all you need is adding
a reference from the NuGet package project:

![Add Reference](/images/AddReference.png)

After adding the reference to `ClassLibrary1`, the NuGet project will now pick
up the contents from `ClassLibrary1`:

![Package Contents](/images/NuGetPackageContents.png)

### What is packaged up?

NuProj will normally include all the `.dll` files that are produced from the
referenced projects. However, there are two special cases:

1. **File coming from NuGet packages**. If the files in the output were coming
from NuGet package, NuProj will add a package dependency instead of adding the
file to the resulting package.

2. **Files that are already packaged by another NuProj project**. Imagine your
solution contains two class library projects and you want to create a NuGet
package for each of them. Now let's also say that the second class library
depends on the first. When building the NuGet package for the second class
library you probably don't want NuProj to include the binary of the first class
library. Instead, you'll probably want NuProj to add a package dependency to the
NuGet project that you built for the first class library. In order to do that,
you need to add reference to the NuProj project of the first class library. The
result would look like the following:

  ![Package Contents](/images/ProjectDependencies.png)

## Adding manual content

Since NuProj is a fully-fledged project type, you can add content as you can
with regular projects. In order to cause the files to be packaged up by NuProj
you'll need to set their `Build Action` to `Content`:

![Build Action](/images/BuildAction.png)

Here are a few examples where adding manual content comes in handy:

* **Readme files**. It's common to have a `README.txt` in the NuGet package.
  In fact, the default template already provides one. More about readme files
  can be found in the [official NuGet documentation](http://docs.nuget.org/docs/creating-packages/creating-and-publishing-a-package#Automatically_Displaying_a_Readme.txt_File_During_Package_Installation).
* **Installation scripts**. Some NuGet packages require custom installation
  scripts. See [official NuGet documentation](http://docs.nuget.org/docs/creating-packages/creating-and-publishing-a-package#Automatically_Running_PowerShell_Scripts_During_Package_Installation_and_Removal)
  for more details.

# Raw MSBuild authoring

In general, the definition of a NuProj file is very similar to the NuGet .nuspec
file - except that it is done via MSBuild. In fact, NuProj generates the .nuspec
file and simply calles NuGet to build the package. Thus, for details you can
still refer to the [official NuGet documentation](http://docs.nuget.org/docs/reference/nuspec-reference).

## Metadata

Most of the metadata is represented via simple MSBuild properties:

```xml
<PropertyGroup>
  <Id>HelloWorld</Id>
  <Version>1.0.0</Version>
  <Title>HelloWorld</Title>
  <Authors>The World</Authors>
  <Owners>$(Authors)</Owners>
  <Description>Hello World</Description>
  <!-- Optional information:
  <ReleaseNotes></ReleaseNotes>
  <Summary>Hello World</Summary>
  <ProjectUrl>http://example.com</ProjectUrl>
  <LicenseUrl>http://example.com/license</LicenseUrl>
  <Copyright>Copyright (c)</Copyright>
  <RequireLicenseAcceptance>False</RequireLicenseAcceptance>
  <DevelopmentDependency>False</DevelopmentDependency>
  <Tags>HelloWorld</Tags>
  -->
</PropertyGroup>
```

This allows defining the metadata in terms of other MSBuild properties (such as
`Owners` is simply defined via the `Authors` property). Also the, properties
can be defined via command line arguments passed to MSBuild.

## Dependencies

Package dependencies are expressed via MSBuild item groups:

```xml
<ItemGroup>
  <Dependency Include="RouteMagic">
    <Version>1.1.6</Version>
  </Dependency>
</ItemGroup>
```

The package ID is passed via the standard `Include` attribute. The version is
specified as a custom metadata element within the item. Also, you can indicate
that the dependency is only there when targeting a certain framework:

```xml
<ItemGroup>
  <Dependency Include="RouteMagic">
    <Version>1.1.6</Version>
    <TargetFramework>net40</TargetFramework>
  </Dependency>
</ItemGroup>
```

Packages marked as development dependencies will be ignored.

```xml
<ItemGroup>
  <Dependency Include="Fody">
    <Version>1.25.0</Version>
    <DevelopmentDependency>true</DevelopmentDependency>
  </Dependency>
</ItemGroup>
```

## Framework assemblies

NuGet supports adding references to framework assemblies as well. You can
specify those via the `FrameworkReference` item:

```xml
<ItemGroup>
  <FrameworkReference Include="System.dll" />
  <FrameworkReference Include="System.Core.dll" />
</ItemGroup>
```

As with dependencies, those can be specific to a certain target framework:

```xml
<ItemGroup>
  <FrameworkReference Include="System.dll" />
  <FrameworkReference Include="System.Core.dll">
    <TargetFramework>net40</TargetFramework>
  </FrameworkReference>
</ItemGroup>
```

## Packaging files

Packaging library files is done as follows:

```xml
<ItemGroup>
  <Library Include="$(BasePath)HelloWorld.dll" />
    <TargetFramework>net40</TargetFramework>
  </Library>
</ItemGroup>
```

The `TargetFramework` metadata is optional. If it's missing, the library will
apply to all platforms. Please note that per convention the path to the file
name is prefixed with the `$(BasePath)` property. This allows the build process
to control where the binaries should be picked up from.

For packaging content files you can use the `Content` item:

```xml
<ItemGroup>
  <!-- Package license.txt into content folder. -->
  <Content Include="$(BasePath)license.txt" />
  <!-- Package all .css files into content\css -->
  <Content Include="$(BasePath)css\**\*.css">
    <TargetPath>css</TargetPath>
  </Content>
  <!-- The following will also rename readme.txt to readme_HelloWorld.txt -->
  <Content Include="$(BasePath)readme.txt">
    <TargetPath>notes\readme_HelloWorld.txt</TargetPath>
  </Content>
</ItemGroup>
```

The `Content` element supports an optional `TargetPath` metadata element. If not
specified, the file will packaged directly into the content folder.

For ultimate control, you can also use the File item which allows you to package
arbitrary files:

```xml
<ItemGroup>
  <File Include="$(BasePath)build.proj">
    <TargetPath>tools\build</TargetPath>
  </File>
</ItemGroup>
```

## Controlling library references

For a given a target platform, NuGet will add references to all the libraries in
the corresponding lib folder. You can override this behavior by using the
`Reference` item:

```xml
<ItemGroup>
  <Reference Include="HelloWorld.dll" />
</ItemGroup>
```

Please note that the reference item doesn't support the `TargetFramework`
metadata. Instead, NuGet will only add the references if the file is also
packaged in the corresponding lib folder. If it's not the reference is simply
ignored. In other words, the target framework is already controlled by the fact
that the file might or might not be in the corresponding lib folder.

### Suppressing references

When a NuPkg packages up references that you do not want to be packaged up,
you may suppress the packaging of that reference by setting `CopyLocal=false`
on that reference within Visual Studio (this becomes `Private=false` in MSBuild).
Alternately, if you need `CopyLocal=true` semantics for other reasons but still
wish to suppress packaging into NuPkg, you can add special metadata to the
reference yourself:

```xml
<ItemGroup>
  <Reference Include="DoNotPackageThis.dll">
    <ExcludeFromNuPkg>true</ExcludeFromNuPkg>
  </Reference>
</ItemGroup>
```

# Checked-in toolkit

On the build machine, you shouldn't install NuProj. Instead, you should restore
the [NuGet package that provides the build server support](http://www.nuget.org/packages/NuProj).

In order for MSBuild to find the `NuProj.targets` that NuProj files depend on
you need to override the `NuProjTargetsPath` property:

```xml
<PropertyGroup>
    <MyNuProjPath>$(MyCheckinRoot)packages\NuProj.0.9.2\tools\</MyNuProjPath>
    <NuProjTargetsPath>$(MyNuProjPath)NuProj.targets</NuProjTargetsPath>
    <!--
    <NuProjTasksPath>$(MyNuProjPath)NuProj.Tasks.dll</NuProjTasksPath>
    <NuGetToolPath>$(MyNuProjPath)</NuGetToolPath>
    <NuGetToolExe>NuGet.exe</NuGetToolExe>
    -->
</PropertyGroup>
```

Optionally, you can also chose a different layout and override the other
properties as well.
