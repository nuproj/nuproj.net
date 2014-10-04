---
title: Documentation
layout: default
---

# Getting started

NuProj provides an MSBuild-based approach to create NuGet packages (.nupkg). The
build projects are called .nuproj files and are regular MSBuild projects.

You can create new a .nuproj file by using the template that we install into
Visual Studio. Invoke the menu command **File | New | File...** or simply press
<kbd>Ctrl + N</kbd> and select **NuGet Package MSBuild Project**.

In general, the definition of a NuProj file is very similar to the NuGet .nuspec
file - except that it is done via MSBuild. In fact, NuProj generates the .nuspec
file and simply calles NuGet to build the package. Thus, for details you can
still refer to the [official NuGet documentation](http://docs.nuget.org/docs/reference/nuspec-reference).

# Authoring

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

## Framework Assemblies

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

## Packaging Files

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

## Controlling Library References

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

# Checked-in Toolkit

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